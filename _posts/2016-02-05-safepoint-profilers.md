At QCon 2015 last November, we saw [this presentation](http://www.infoq.com/presentations/profilers-hotspots-bottlenecks) by Nitsan Wakart talking about the limitations of sampling profilers (most of them).  The basic idea is that the normal JVM profiling API can only sample at “safepoints” which are basically method calls, loops and certain other places where the VM is basically halted while it records its state.  Because of hotspot optimization and code inlining, it is quite common that method calls become invisible to the profiler and the code locations reported by the profiler are only approximate.  This makes their results difficult and sometimes misleading to interpret.

It turns out that the latest versions of Oracle’s JVM have the ability to do non-safepoint profiling, with the right flags enabled.

```
-XX:+UnlockCommercialFeatures -XX:+FlightRecorder -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints
```

The difference in fidelity is hard to demonstrate, but as an example, check out these two traces of the same thread executing similar workloads.  The first was taken using normal safepoint profiling.  The second was taken with the non-safepoint profiling flag.

![Safepoint Profile](images/safepoint-profile.png)

![Non-Safepoint Profile](images/non-safepoint-profile.png)
