---
layout: default
lang: EN
title: Cache effects in hybrid multicore processors
date:   2026-01-11
tag: english technology
---

## Cache Effects in hybrid multicore processors
Legacy software for symmetric multicore processors (SMP) was typically written with a small number of identical cores in mind. As the number of cores scale to beyond `20` cores on user workstations and much larger numbers available in cloud computing, some of the software design patterns might not scale that well with the increased compute resources. Moreover a significant number of multicore deployment today is a hybrid configuration where some CPUs are optimized for power efficiency while others can target higher performance, this *might* also result in an unknown environment for legacy software. Hybrid or heterogeneous compute model is pretty common now as seen with various ARM [big.Little](https://www.arm.com/technologies/big-little) processors or Intel i7 Raptor/Alder Lake [architecture]( https://en.wikipedia.org/wiki/Raptor_Lake)


This note describes cache performance evaluations with a single threaded CPU bound function made to periodically run across various cores on a hybrid multicore system. Measurements were conducted on an Intel 13th Gen Intel(R) Core(TM) i7-13850HX processor using GNU Linux. The program was built with `O3` optimization level using Clang CPP compiler. Below snippet shows the function under evaluation, for brevity; main thread setting up this function to run with highest pthread priority under FIFO scheduling and someother boilerplate is removed from the code listing.  The last conditional in the function guards against any out-of-order execution ill-effects during longer execution cycles.

```cpp
using namespace std;


// 2D buffer containing counters
static constexpr size_t WIDTH = 1024;
static constexpr size_t HEIGHT = 256;
static uint64_t countingBuffer[HEIGHT][WIDTH];

static void* count_then_hop_freerun(void* maxCoresArg)
{

   if (maxCoresArg == nullptr) {
       cerr << "Invalid input argument for thread function" << endl;
       exit(1);
   }
   const size_t maxCores = *reinterpret_cast<size_t *>(maxCoresArg);
   const size_t LastCol = WIDTH - 1;
   const size_t LastRow = HEIGHT - 1;
   while (countingBuffer[LastRow][LastCol] <= numeric_limits<uint64_t>::max()) {
       uint64_t prev = countingBuffer[LastRow][LastCol];

       // Inform scheduler about preference to run on a different core
       set_affinity(nextCore);
       usleep(1);
       for (size_t i = 0; i < WIDTH; i++) {
           for (size_t j = 0; j < HEIGHT; j++) {
               countingBuffer[j][i]++;
           }
       }
       if (countingBuffer[LastRow][LastCol] != prev + 1) {
           cout << "Last counter=" << countingBuffer[LastRow][LastCol] << " Currently executing on Core:" << get_current_cpu() << " NOT CONSISTENT" << std::endl;
           exit(1);
       }
   }
   exit(0);
}
```

The first set of measurements establishes a baseline without user triggered core migration. Nothing surprising here; one could see Linux scheduling the program on a smaller ATOM CPU for the execution, which is completely occupied running the counting buffer loop. Very little time (0.17%) as seen with associated flamegraph is spent in kernel mode.

```bash
perf stat -B -e cache-references,cache-misses,cycles,instructions,branches,faults,migrations ./busy_loop_nohopp
Number of available CPU Cores=28

 Performance counter stats for './busy_loop_nohopp':

        37,528,998      cpu_atom/cache-references/ 
     <not counted>      cpu_core/cache-references/                                   (0.00%)
           380,339      cpu_atom/cache-misses/           #    1.01% of all cache refs 
     <not counted>      cpu_core/cache-misses/                                       (0.00%)
   943,288,252,741      cpu_atom/cycles/                                                      
     <not counted>      cpu_core/cycles/                                             (0.00%)
   136,962,394,959      cpu_atom/instructions/           #    0.15  insn per cycle            
     <not counted>      cpu_core/instructions/                                       (0.00%)
    19,636,695,283      cpu_atom/branches/                                                    
     <not counted>      cpu_core/branches/                                           (0.00%)
            639      faults                                                                
            6      migrations                                                            

    295.627415496 seconds time elapsed

    295.517465000 seconds user
    0.002999000 seconds sys
```

![BusyLoopNoHopp]({{ site.baseurl}}/assets/images/busy_loop_nohopp.svg)

Same function when run with forced code migration is shown below.
```cpp
static void* count_then_hop_freerun(void* maxCoresArg)
{

   if (maxCoresArg == nullptr) {
       cerr << "Invalid input argument for thread function" << endl;
       exit(1);
   }
   const size_t maxCores = *reinterpret_cast<size_t *>(maxCoresArg);
   const size_t LastCol = WIDTH - 1;
   const size_t LastRow = HEIGHT - 1;
   while (countingBuffer[LastRow][LastCol] <= numeric_limits<uint64_t>::max()) {
       uint64_t prev = countingBuffer[LastRow][LastCol];
       int nextCore = get_current_cpu();
       if (nextCore < maxCores / 2) {
           nextCore = maxCores - 1 - nextCore;
       } else if (nextCore == maxCores / 2) {
           nextCore = 0;
       } else {
           nextCore = maxCores - nextCore;
       }
       // Inform scheduler about preference to run on a different core
       set_affinity(nextCore);
       usleep(1);
       for (size_t i = 0; i < WIDTH; i++) {
           for (size_t j = 0; j < HEIGHT; j++) {
               countingBuffer[j][i]++;
           }
       }
       if (countingBuffer[LastRow][LastCol] != prev + 1) {
           cout << "Last counter=" << countingBuffer[LastRow][LastCol] << " Currently executing on Core:" << get_current_cpu() << " NOT CONSISTENT" << std::endl;
           exit(1);
       }
   }
   exit(0);
}
```

```bash
Performance counter stats for './busy_loop_2d_hopp':

     9,357,838,372      cpu_atom/cache-references/                                              (44.05%)
    17,403,877,538      cpu_core/cache-references/                                              (55.95%)
        22,316,030      cpu_atom/cache-misses/           #    0.24% of all cache refs           (44.05%)
         9,593,899      cpu_core/cache-misses/           #    0.06% of all cache refs           (55.95%)
   870,509,131,286      cpu_atom/cycles/                                                        (44.05%)
   898,005,671,015      cpu_core/cycles/                                                        (55.95%)
   125,686,755,390      cpu_atom/instructions/           #    0.14  insn per cycle              (44.05%)
   132,711,652,352      cpu_core/instructions/           #    0.15  insn per cycle              (55.95%)
    18,140,905,879      cpu_atom/branches/                                                      (44.05%)
    19,184,599,056      cpu_core/branches/                                                      (55.95%)
               641      faults                                                                
           265,975      migrations                                                            

     391.024668087 seconds time elapsed

     372.148367000 seconds user
      15.468433000 seconds sys

```
![BusyLoop2DNoHopp]({{site.baseurl}}/assets/images/busy_loop_2d_hopp.svg)

Here we note a couple of interesting observations

- Firstly, both smaller and performance cores are now indeed engaged.
- As expected we have a high number of migrations and page faults
- The contribution of system calls is more visible `(0.73%)` due to forced rescheduling via sleep and affinity routines.
- The percentage of cache misses actually decreased on smaller cores with a marginal increase on performance cores. This is noteworthy, as there was a manyfold increase in the actual number of cache references, most likely from kernel calls, which were still served from the cache. Here we also see advantages of cache coherence where an indirect miss from L1 private cache can still be served from higher level of L2/L3 shared caches.


Now we are ready to see the cache impact as the buffer size grows.
![CacheMiss]({{ site.baseurl}}/assets/images/cache_misses_percentage.png)

We can see that cache coherence works quite well as long as the buffer size is less than about half of total cache size (`30MB` on this system). We also note that once the buffer size is comparable to actual cache available on the system smaller cores tend to spend much more time waiting for memory compared to performance cores. On performance most of the cache overhead originates from system calls, something which can be clearly seen upon running `64MB` counting buffer configuration with affinity and sleep routines.

```bash
    19,673,428,160      cpu_atom/cache-references/                                              (100.00%)
    29,602,924,301      cpu_core/cache-references/                                              (0.00%)
    19,403,460,490      cpu_atom/cache-misses/           #   98.63% of all cache refs           (100.00%)
     5,254,372,506      cpu_core/cache-misses/           #   17.75% of all cache refs           (0.00%)
   968,368,314,135      cpu_atom/cycles/                                                        (100.00%)
   520,745,817,948      cpu_core/cycles/                                                        (0.00%)
    36,403,854,583      cpu_atom/instructions/           #    0.04  insn per cycle              (100.00%)
   263,996,281,853      cpu_core/instructions/           #    0.51  insn per cycle              (0.00%)
     5,276,084,628      cpu_atom/branches/                                                      (100.00%)
    45,914,526,678      cpu_core/branches/                                                      (0.00%)
            32,894      faults                                                                
                25      migrations                                                            

     327.812572224 seconds time elapsed

     327.588185000 seconds user
       0.060970000 seconds sys
```

In summary, where possible legacy code's well-intentional tinkering with OS scheduling decisions should be scrapped on heterogeneous computing. Also as long as the user programs abide by the memory model, parallel access to cached data without loss of functional correctness can efficiently be taken care of by the hardware implemented cache coherency protocols. 
