Now I have everything. Let me build a comprehensive interactive checklist diagram.Tick items off as you apply them. Now let me walk through every layer in depth.

---

## Layer 1 — BIOS / Hardware (Do This First)

The single most damaging thing to latency is the CPU quietly sleeping to save power and then taking 50–200 µs to wake up. Setting the CPU governor to "performance" prevents frequency throttling, ensuring consistent core speeds. Disabling Hyper-Threading eliminates resource contention and unpredictable latency, maintaining isolation and cache availability per core.

The NUMA topology is critical and most people skip it. The `lstopo` utility determines PCIe locality — in a dual-socket server, your NIC's PCIe lanes connect to one specific NUMA node. If your app runs on the other node, every packet crosses the QPI/UPI interconnect, adding 60–200 ns of invisible latency. Run `lstopo` first, find which NUMA node your SFN8522 is on, and pin everything — CPUs, memory, app — to that node.

---

## Layer 2 — Linux Kernel Cmdline

This is where most latency comes from on a vanilla Linux server. The kernel has timers, scheduler ticks, RCU callbacks, and workqueue threads that randomly interrupt your hot core every few milliseconds. You surgically remove all of them from your hot cores.

The `isolcpus` + `nohz_full` + `rcu_nocbs` combination is the standard recipe: combining `rcu_nocb_poll`, `rcu_nocbs=2-95`, `nohz_full=2-95`, `kthread_cpus=0,1`, `irqaffinity=0,1`, and `isolcpus=managed_irq,domain,2-95` produces a system where cores 2–95 are almost completely isolated from kernel interference.

Your `/etc/default/grub` line ends up looking like:

```bash
GRUB_CMDLINE_LINUX="isolcpus=managed_irq,domain,2-15 nohz_full=2-15 nohz=on \
  rcu_nocbs=2-15 rcu_nocb_poll kthread_cpus=0,1 irqaffinity=0,1 \
  intel_idle.max_cstate=0 processor.max_cstate=0 \
  default_hugepagesz=2M hugepages=2048 iommu=pt"
```

Then at runtime, disable Transparent Hugepages (which causes latency spikes during memory compaction) with `echo never` to both the `enabled` and `defrag` sysfs files, and use `mlockall()` in your application code to lock all pages in RAM and prevent page faults.

---

## Layer 3 — Your Solarflare SFN8522 (The Key Layer)

The SFN8522-Onload unlocks sub-microsecond latency at both Layer 2 and TCP/UDP via Onload — a fully RFC-compliant kernel-bypass userspace TCP/IP stack requiring no application modification.

But for the absolute minimum latency, you have three modes in increasing order of performance:

**Mode A — Onload with `LD_PRELOAD`** (easiest): Your existing socket-based C++ code runs unchanged. Just prepend `onload ./myapp`. Onload achieves kernel bypass by implementing the network stack in userspace and using an `LD_PRELOAD` to overwrite network syscalls of the target program. Latency roughly halved vs raw kernel sockets.

**Mode B — TCPDirect** (medium): Rewrite your socket calls to use `zsockets` — a simplified API. TCPDirect is an ultra-low latency networking stack that employs kernel bypass similar to Onload but achieves even lower latency.

**Mode C — ef_vi** (lowest possible): Bypass everything. The `ef_vi` API grants the application direct access to the Solarflare network adapter datapath, delivering the lowest latency and reduced per-message processing overhead. It is for applications that want the very lowest latency send/receive API and do not require a POSIX socket interface.

Here's the minimal ef_vi receive loop in C++:

```cpp
#include <etherfabric/vi.h>
#include <etherfabric/pd.h>

ef_driver_handle dh;
ef_pd           pd;
ef_vi           vi;
ef_memreg       mr;

// Setup (once at startup)
ef_driver_open(&dh);
ef_pd_alloc(&pd, dh, ifindex, EF_PD_DEFAULT);
ef_vi_alloc_from_pd(&vi, dh, &pd, dh,
    -1,           // evq_capacity (-1 = default)
    512,          // rxq_capacity
    512,          // txq_capacity
    NULL, -1,     // no separate evq
    EF_VI_FLAGS_DEFAULT);

// Pre-post receive buffers (do this N times)
// buf is a hugepage-backed DMA buffer
ef_vi_receive_post(&vi, dma_addr, buf_id);

// Hot loop — zero syscalls
ef_event evs[16];
while (true) {
    int n = ef_eventq_poll(&vi, evs, 16);  // spin poll
    for (int i = 0; i < n; i++) {
        if (EF_EVENT_TYPE(evs[i]) == EF_EVENT_TYPE_RX) {
            int id  = EF_EVENT_RX_RQ_ID(evs[i]);
            int len = EF_EVENT_RX_BYTES(evs[i]);
            uint8_t* pkt = get_buf(id);  // your buffer lookup
            process_packet(pkt, len);    // your strategy logic
            ef_vi_receive_post(&vi, dma_addr_of(id), id);  // repost
        }
    }
}
```

For transmit, use PIO (Programmed IO) for small frames — it copies the packet directly into NIC SRAM and sends it without DMA:

```cpp
// TX via PIO — lowest possible transmit latency for small packets (<80 bytes)
ef_pio pio;
ef_pio_alloc(&pio, dh, &pd, -1, &vi);
ef_pio_link_vi(&pio, dh, &vi);

// In hot path:
memcpy(pio_buf, order_payload, order_len);
ef_vi_transmit_pio(&vi, 0, order_len, tx_id);
// No DMA, no kernel, packet hits wire in ~100 ns
```

CTPIO (Cut-Through Programmed IO) can also be enabled, which bypasses the main adapter datapath entirely. It's controlled via `EF_CTPIO=1` and `EF_CTPIO_MODE=ct` for cut-through mode.

---

## Layer 4 — The C++ Application Rules

On top of all the kernel work, your hot thread must follow these rules or you'll throw away all the gains:

No memory allocation in the hot loop — ever. No `new`, no `std::vector::push_back`, no `std::string`. Pre-allocate everything at startup using a pool allocator backed by hugepages. No virtual function calls (vtable lookup + branch misprediction). No exceptions (they use a zero-cost mechanism that still has overhead in the `throw` path). Compile with `-O3 -march=native` — this enables AVX2/AVX-512 auto-vectorization and eliminates function call overhead through inlining. Configure your application to lock memory and prevent page faults by using `mlockall()` in C code.

The measurement tool for all of this is `cyclictest` for kernel jitter, and your own `rdtsc`-based timestamping at the NIC receive → order send boundaries to measure actual end-to-end latency.
