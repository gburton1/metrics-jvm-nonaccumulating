metrics-jvm-nonaccumulating
---
### Description
This metric set class alters the behavior of the Dropwizard's GarbageCollectorMetricSet, which takes readings of Garbage Collection (GC) counts and times that are cumulative over the life of the JVM. This metric set uses gauges that represent data for a specific time interval, rather than all time. It also adds an extra gauge for GC throughput, which is a convenient summary metric for JVM's.

### How does it work?
The NonAccumulatingGarbageCollectorMetricSet is a wrapper around GarbageCollectorMetricSet. To calculate readings for a specific time interval, this metric set needs to maintain state for previous values and non-accumulated values. To do it in a thread safe way (allowing multiple Dropwizard Reporters), this metric set runs a background process in a scheduled thread that updates and stores gauge readings from the GarbageCollectorMetricSet at user-specified time intervals. When Reporters retrieve values from the gauges, the stored readings are read (not modified), rather than calling the underlying gauges of the GarbageCollectorMetricSet.

Multiple Dropwizard Reporters can still be used to read the metrics--it makes sense to configure those reporters to report on the same time interval as this library, but it is not required. If you schedule a reporter more frequently than the time interval of the library, then you will get repeated samples. If you a schedule a reporter less frequently, you will miss some data samples.

### Garbage Collector Throughput (GC throughput)
GC throughput is derived from the readings of time spent by all garbage collectors in the current time interval. Recall that the JVM uses a different garbage collector for the young and old generation spaces of the heap. 

For example, for an interval of 1 minute, we have 60,000 ms of total time. If we spent 200 ms on minor GC's and 100 ms on major GC's, then 59,700 ms were available to the application. 59,700 / 60,000 = 99.5% GC throughput. While there are other overhead operations of the JVM that also take away CPU time from the application, garbage collection represents the vast majority of the overhead. Therefore, GC throughput is a value between 0.0 and 100.0 that represents proportion of CPU time available to the application (not used for garbage collection).

### Usage
This is a Spring configuration file that shows the basic setup to create this metric set, and register all the underlying metrics in the set.

    @Bean
    @ConditionalOnMissingBean(MetricRegistry.class)
    public MetricRegistry metricRegistry() {
        return new MetricRegistry();
    }

    @Value("${graphite.reporting.interval:60000}")
    private long interval;

    @PostConstruct
    public void jvmMetrics() {
        registerAll("JvmStats.GarbageCollector", new NonAccumulatingGarbageCollectorMetricSet(new GarbageCollectorMetricSet(), interval));
    }

    private void registerAll(String metricPrefix, MetricSet metrics) {
        for (Map.Entry<String, Metric> entry : metrics.getMetrics().entrySet()) {
            if (entry.getValue() instanceof MetricSet) {
                registerAll(MetricRegistry.name(metricPrefix, entry.getKey()), (MetricSet) entry.getValue());
            } else {
                metricRegistry().register(MetricRegistry.name(metricPrefix, entry.getKey()), entry.getValue());
            }
        }
    }

    // add Dropwizard Reporters (Graphite, etc.)
