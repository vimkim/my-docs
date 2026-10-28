# Coverage of Brendan Gregg performance material in our Linux survey

Review date: **2026-10-28**. Reviewed document: [Linux DB and systems performance survey](2026-10-28-linux-db-performance-survey.md), revision `b94f04a`.

**Our survey covers most measurement capabilities in the linked talk and adds many specialist tools. Its largest gap is a systematic investigation method.** A reader can select tools, but still needs guidance to inventory resources, reject competing explanations, identify the benchmark limiter, and justify a change.

The linked source is the 72-slide **PerconaLive 2016 Linux Systems Performance presentation**. Its six subjects provide the main comparison below. The book comparison uses *Systems Performance*, second edition (2020), as the assumed edition. It uses the publisher’s contents, index and sample chapter. It does not claim a full-text review of the complete book. Selected later Gregg materials prevent false claims of novelty. [Presentation and transcript](https://www.brendangregg.com/Slides/Percona2016_LinuxSystemsPerf/), [publisher’s sample](https://www.informit.com/content/images/9780136820154/samplepages/9780136820154_Sample.pdf), [author’s edition description](https://www.brendangregg.com/systems-performance-2nd-edition-book.html).

“Covered” means that the survey explains the purpose and material limits of a capability. A tool name alone does not establish teaching depth. “Partial” means that the survey supplies some ingredients but does not explain the complete method. The omission decisions below are recommendations for the current survey. They do not establish why its author originally omitted a topic.

## Comparison with the linked presentation

| Presentation subject | Slides | Survey evidence | Assessment |
| --- | --- | --- | --- |
| Observability | 5–21 | Sections 3.1–3.7, especially tools 2–4, 9–11, 19, 29, 33, 46–48 | Broad coverage. Missing initial host triage and kernel slab accounting matter more than another general monitoring tool. |
| Methodologies | 22–32 | Symptom questions in section 1, selection priorities in section 8, DB context in tool 49 | Partial. No explicit USE checklist or procedure for eliminating resource hypotheses. |
| Benchmarking | 33–39 | Sections 4, 6 and 7 | Strong controls and reproducibility. Partial explanation of how to prove the limiting component while the workload runs. |
| Profiling | 40–51 | Tools 9–12, 17–18, section 5.2, interpretation cautions in section 6 | Strong capability coverage. FlameGraph generation and a worked interpretation remain absent. |
| Tracing | 52–65 | Tools 13–16, 19–27, 30–33, examples in sections 5.3–5.5 | Strong capability coverage. The relationship between event sources, collection frameworks and front ends remains implicit. |
| Tuning | 66–68 | Experimental prerequisites in section 2, knobs mentioned in tools 34 and 47, recording rules in section 7 | Mostly outside the measurement guide. Add a short experiment procedure if the guide should also support tuning. |

Slide references point to the [original presentation](https://www.brendangregg.com/Slides/Percona2016_LinuxSystemsPerf/). The assessments describe the local survey, rather than a percentage of book content.

## Important gaps worth addressing

### Systematic resource checks and error checks

**Missing method, partly available measurements.** The survey covers utilization, pressure and queues in many separate rows. It does not require a complete resource inventory or an explicit error check for each resource.

Add a small USE matrix: resource, utilization, saturation, errors, result, and unresolved evidence. Include CPU, memory capacity, storage, network, interconnects, and relevant software limits. Record “unknown” when a measurement is unavailable. This makes the investigation auditable and reduces the chance of overlooking a resource because its tool is unfamiliar. [Gregg’s USE method](https://www.brendangregg.com/usemethod.html).

For DB development, extend the inventory to worker pools, connection capacity, file descriptors and user locks. Reuse the survey’s cgroup, scheduler and lock measurements. Add resource-specific capacity and error counters where necessary. The checklist should require systematic checks. [USE software resources](https://www.brendangregg.com/usemethod.html#Software).

### Initial host triage including kernel errors

**Missing procedure.** Section 5.1 checks the platform and profiling prerequisites. It does not first check whether the host reports OOM events, storage failures, or another error that changes the investigation.

Add a compact initial pass over kernel messages, CPU balance, run queues, available memory, swap, device latency and network errors. Most required tools already appear in the survey. Kernel messages and a readable memory summary need explicit entries. Error checks belong before expensive profiling. Gregg’s Linux USE checklist includes kernel messages, resource errors and capacity limits. [Linux USE checklist](https://www.brendangregg.com/USEmethod/use-linux.html).

### Active benchmarking and an evidenced limiter

**Partial coverage.** The survey already requires client utilization, server counters, correct cache conditions, repeats and collection overhead checks. It does not require a benchmark result to explain what limited throughput or latency.

Add a result field for the suspected limiter and its evidence. Examples include one saturated worker, a full client CPU, bandwidth exhaustion, CPU quota throttling, or storage latency. A controlled follow-up experiment should test that explanation. This extends the existing recording template instead of adding another benchmark program. [Gregg’s active benchmarking method](https://www.brendangregg.com/activebenchmarking.html).

### A complete CPU profile interpretation example

**Partial coverage.** Section 5.2 reaches `perf report` and `perf annotate`. Tool 11 explains graph width and warns that the horizontal axis is not time. The survey does not show how to produce a FlameGraph or use ancestry to explain a hot path.

Add the existing `perf.data` → `perf script` → folded stacks → SVG sequence. Include one small C++ example that separates self cost, inclusive cost, call frequency and cost per completed operation. Preserve the existing cautions about absolute cost and request latency. [Gregg’s CPU FlameGraph explanation and generation instructions](https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).

### Kernel memory and file system latency

**Missing kernel memory view.** The survey explains process RSS/PSS, heaps, allocator fragmentation, page cache and memory pressure. It does not explain slab caches. Add `/proc/meminfo` slab fields and `slabtop -o` as a short branch for memory growth that process heaps do not explain. `slabtop` reports kernel caches, with accounting limitations that need care. It remains a current tool. [slabtop manual](https://man7.org/linux/man-pages/man1/slabtop.1.html).

**Partial file system view.** Tools 30–32 distinguish block events from file system and syscall boundaries. A concrete `ext4slower` or matching file system latency example would make that distinction usable. A slow file operation may require investigation even when block-device averages look acceptable. Add this after the existing fsync example, rather than treating block latency as its substitute. [BCC ext4slower source](https://github.com/iovisor/bcc/blob/master/tools/ext4slower.py).

### Event selection and controlled tuning

**Partial explanation.** The tracing table lists frameworks, interfaces, event sources and ready-made programs together. Add a small map showing PMU/software events, tracepoints and dynamic probes feeding collectors and reports. Then explain why a request timer, syscall interval and block interval answer different questions. The existing fsync example can serve as the worked case. [Gregg’s perf event taxonomy](https://www.brendangregg.com/perf.html#Events).

**Optional scope extension.** The survey describes several tunable conditions without a tuning procedure. If tuning belongs in scope, add an experiment procedure. Record the current setting and state a hypothesis. Change one condition and repeat the same workload. Check correctness and tail latency, then retain or revert the change. This recommendation builds on the survey’s own experimental controls. It does not require copying the presentation’s historical settings.

## Additional gaps compared with the book and later materials

The survey is a tool selection guide. It does not provide the book’s treatment of models, architecture, diagnosis and tuning. These are differences in teaching depth, even where the survey names a relevant tool. [Publisher’s chapter outline](https://www.informit.com/store/systems-performance-9780136820154).

| Topic | Book reference | Current coverage and value for C++ and DB work |
| --- | --- | --- |
| Scaling and queueing models | Sections 2.6–2.7 | Missing. Explain Amdahl’s law, the Universal Scalability Law, queueing and capacity planning. Use them to interpret throughput versus worker count and rising latency near saturation. |
| Service and thread analysis | Sections 2.5.10 and 5.4.5 | Partial. The survey records errors, latency, off-CPU stacks and run-queue delay. Add the RED method and a thread-state procedure that distinguishes useful work, runnable delay, blocking and idle workers. |
| Subsystem models and architecture | Chapters 3 and 6–10 | Limited to interpretation cautions. Add short explanations of CPU pipelines and memory hierarchy, virtual memory and reclaim, the file I/O path, and the network path. These explain why measurements differ across layers. |
| Tuning and network experiments | Sections 6.9–10.8 | Mostly outside scope. Add selected experiments only when a diagnosis needs them, such as CPU placement or network capacity tests. A complete tuning handbook is optional. |
| Cloud and virtualization diagnosis | Chapter 11 | Partial. PMU visibility and cgroups appear. Hypervisor overhead, shared-host interference and instance limits need more explanation if the target DB runs in a VM or cloud. |

These references follow the [published contents](https://www.informit.com/store/systems-performance-9780136820154). They establish the book’s subject coverage, not the detail of every example.

Two additions would make the survey’s existing measurements more useful:

- **Separate idle workers from request-related waits.** Large aggregate off-CPU time can reflect workers waiting for work. Add request context before ranking waits as bottlenecks. [Gregg’s thread-state method](https://www.brendangregg.com/tsamethod.html), [off-CPU analysis](https://www.brendangregg.com/offcpuanalysis.html).
- **Show distributions over time.** The survey records percentiles and histograms, but does not teach latency heat maps or FlameScope. These help isolate short phases and periodic interference. Use FlameScope on collected profiles without launching a GUI on the remote server. [Heat maps](https://www.brendangregg.com/heatmaps.html), [FlameScope](https://www.brendangregg.com/flamescope.html).

Distributed request tracing is another conditional gap. It matters when client, proxy and database work must be correlated across processes or machines. Local kernel tracing alone does not provide that application request context. [Book section 5.4.8 in the publisher’s contents](https://www.informit.com/store/systems-performance-9780136820154).

## Omissions that are reasonable to retain

| Omitted item | Classification | Reason and present coverage |
| --- | --- | --- |
| `netstat` command recipes | Legacy interface with replacements | Its manual calls it mostly obsolete. Tool 46 already covers `ss` and `nstat`. Routing and interface inspection can use `ip route` and `ip -s link`. [netstat manual](https://man7.org/linux/man-pages/man8/netstat.8.html). |
| Separate `perf-tools` recipes for `iosnoop`, `iolatency` and function counts | Alternative implementations, not a missing concept | Tools 20–25 and 30 cover the measurement capabilities with ftrace, BCC and bpftrace. The original collection describes implementation workarounds and unsupported development tools. It can remain useful where dependencies or kernel support favor it. [perf-tools README](https://github.com/brendangregg/perf-tools). |
| `pcstat` as another page residency command | Optional alternative | Tool 33 already explains file cache state through `cachestat(2)`. `pcstat` uses `mincore(2)` and can remain a useful alternative. BCC `cachestat` is a different activity estimator. [pcstat](https://github.com/tobert/pcstat), [cachestat syscall](https://man7.org/linux/man-pages/man2/cachestat.2.html). |
| `uptime`, `top` and `htop` tutorials | Basic convenience interfaces | The survey already has more targeted process and CPU tools. A brief initial triage mention is sufficient. Their omission does not make them deprecated or useless. |
| The old `free` output layout | Historical presentation detail | Teach the current `available` field instead of reproducing the older buffers/cache columns. Memory availability remains useful for initial triage. [free manual](https://man7.org/linux/man-pages/man1/free.1.html). |
| Ubuntu Trusty and Xen tuning values from slides 67–68 | Historical environment-specific configuration | Do not treat those settings as a portable current baseline. Modern block queues and scheduler choices require current device and kernel evidence. NUMA balancing, huge pages and writeback remain valid experiment subjects. [Current blk-mq documentation](https://docs.kernel.org/block/blk-mq.html). |
| Debugfs tracing path and historical kernel filenames | Interface documentation changed | The survey already says tracefs and links current documentation. `/sys/kernel/tracing` is the current documented mount location. The debugfs path remains available for compatibility. [ftrace filesystem documentation](https://docs.kernel.org/trace/ftrace.html). |
| Netflix dashboard names, historical demonstrations and profiler subcommand inventories | Context or optional breadth | They do not need one-for-one replacements in a DB developer’s measurement guide. Add a concrete capability only when its absence blocks an investigation. |

These classifications separate three cases: a superseded interface, an alternative implementation, and material outside the guide’s scope. They do not label the entire 2016 presentation obsolete.

## What our survey adds beyond the 2016 talk

The following comparison concerns the complete talk, including its diagrams. It does not establish absence from every book, talk or article Gregg published.

| Additional coverage in our survey | Survey location | What the survey contributes |
| --- | --- | --- |
| PSI and explicit cgroup v2 accounting | Tools 5–6 | Resource pressure, throttling, group boundaries and writeback attribution caveats. |
| bpftrace and libbpf CO-RE | Tools 23–24, section 5.4 | A current script example and a path to reusable BPF collectors. BPF and BCC themselves already appear in the talk. |
| DAMON and rtla | Tools 28 and 35 | Working set sampling and dedicated OS noise or timer latency investigation. |
| File `cachestat(2)` and `io_uring` tracepoints | Tools 32–33 | File cache state and asynchronous I/O event correlation. The BCC tool named `cachestat` already appears in the talk. |
| Detailed coherence, microarchitecture and hardware tracing options | Tools 12, 17, 40–44 | Top-down analysis, `perf c2c`, PCM, VTune, Intel PT, AMD uProf/IBS and Arm SPE. The talk already introduces PMCs and lists `perf mem`. |
| Heap and execution simulation tools | Tools 36–39 | heaptrack, jemalloc profiling, Massif, Callgrind and Cachegrind, with warnings about measurement distortion. |
| Named DB workload generators and internal metrics | Tool 49, section 4 | fio, pgbench/sysbench, YCSB, microbenchmarks and DB plans or wait statistics, linked to OS evidence. |
| Detailed experiment and collection controls | Sections 2, 5–7 | Binary identity, measurement boundaries, event loss, multiplexing, matched work, open/closed-loop behavior and repeatable A/B records. |

Several apparently newer entries are **already in the 2016 diagrams**: LTTng, SystemTap, `ss`, `turbostat`, `numastat`, `ethtool` and `blktrace` appear on slide 7. The slide 65 BCC map includes `offcputime`, `runqlat`, `biolatency`, `biosnoop`, `cachestat`, `funclatency` and memory tracing. They are overlap, even when the transcript’s text extraction omits them. [Observability diagram](https://www.brendangregg.com/Slides/Percona2016_LinuxSystemsPerf/Percona2016_LinuxSystemsPerf_007.jpg), [BCC diagram](https://www.brendangregg.com/Slides/Percona2016_LinuxSystemsPerf/Percona2016_LinuxSystemsPerf_065.jpg).

## Additions beyond the 2020 book

Absence from the 2016 talk does not establish absence from Gregg’s later work. The following corrections matter when describing the survey’s contribution.

| Apparent addition | Evidence in later Gregg material | Conclusion |
| --- | --- | --- |
| bpftrace | A main subject of the 2019 *BPF Performance Tools* book. [Author’s book page](https://www.brendangregg.com/bpf-performance-tools-book.html). | Additional to the linked talk, already part of his later teaching. |
| libbpf, BTF and CO-RE | Gregg explains them directly in his [2020 BPF binaries article](https://www.brendangregg.com/blog/2020-11-04/bpf-co-re-btf-libbpf.html). | Additional to the talk, not an addition beyond his materials. |
| PSI, cgroup v2 and io_uring | The second-edition sample describes them in section 3.4.1, pages 118–119. [Publisher’s sample](https://www.informit.com/content/images/9780136820154/samplepages/9780136820154_Sample.pdf). | Already in the book. The survey adds a compact selection guide and current measurement caveats. |
| `perf c2c`, `perf mem`, VTune, jemalloc and Valgrind | The sample’s index lists these names. Section 3.4.1 also describes c2c. [Publisher’s sample and index](https://www.informit.com/content/images/9780136820154/samplepages/9780136820154_Sample.pdf). | Do not claim these tool families are absent. Index entries do not establish equal treatment of every subtool. |

The clearest additions after the book’s publication are newer kernel interfaces and tools:

| Addition in our survey | Evidence and boundary | Practical contribution |
| --- | --- | --- |
| Mainline DAMON and `damo`, tool 35 | DAMON reached a mainline release in Linux 5.15 in 2021. [Project release history](https://damonitor.github.io/posts/damon_news/#2021). | Access-pattern and working-set observation beyond RSS or heap allocation profiles. |
| `rtla osnoise` and `timerlat`, tool 28 | The tracers appear in [Linux 5.14 documentation](https://kernel.org/doc/html/v5.14/trace/timerlat-tracer.html). The toolset appears in [Linux 5.17 documentation](https://docs.kernel.org/5.17/tools/rtla/index.html). | Dedicated host-noise and timer-latency investigation, distinct from direct query-latency measurement. |
| File `cachestat(2)`, tool 33 | The syscall arrived in Linux 6.5. [Syscall history and semantics](https://man7.org/linux/man-pages/man2/cachestat.2.html). | File-range cache state, including dirty and writeback pages. This differs from Gregg’s earlier BCC cache-activity estimator. |

These facilities extend the book’s published baseline. This review does not establish that Gregg never mentioned them in later material.

The survey also gives named entries to heaptrack, PCM, AMD uProf and Arm SPE. Its treatment of build identity and DB experiment records is useful. These are explicit additions to this guide’s presentation, with no claim that the underlying techniques are unique or absent from all Gregg material.

## Recommended next changes to the survey

1. Add a short investigation procedure before the tool catalogue: symptom and workload, initial triage, USE and thread-state checks, targeted analysis, hypothesis test, final validation.
2. Add an error column and unresolved evidence to the experiment record. Require an evidenced limiter with benchmark results.
3. Complete the FlameGraph example and add one worked C++ interpretation.
4. Add kernel slab accounting and one file system latency example.
5. Add a tracing layer map and a short scaling and queueing explanation. Include a time-based visualization example when intermittent behavior motivates it.
6. Add cloud or distributed tracing material only for those target environments. Keep historical tuning recipes outside the default workflow.

The first two changes address the largest coverage gap. Adding more tool names before these methods would increase breadth without teaching the reader how to reach a defensible conclusion.
