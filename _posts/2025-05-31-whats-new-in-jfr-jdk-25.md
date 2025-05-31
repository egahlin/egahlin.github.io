---
layout: post
title:  "What's new in JFR in JDK 25"
author:  "ErikGahlin"
date:   2025-05-31 23:10:24 +0200
published: true
image:  "/assets/method-tracer-ui.png.png"
tags: [JFR, JDK 25, Event]
---

JDK 25, to be released on [September 16](https://openjdk.org/projects/jdk/25/), will contain three new Java Enhancement Proposals (JEPs) for JFR.

The first, [JEP 518: JFR Cooperative Sampling](https://openjdk.org/jeps/518), reworks how method sampling is done in the [HotSpot JVM](https://wiki.openjdk.org/display/HotSpot). Stack walking now only happens from a [safepoint](https://openjdk.org/groups/hotspot/docs/HotSpotGlossary.html#safepoint), but without safepoint bias that traditional [JVM TI](https://docs.oracle.com/en/java/javase/22/docs/specs/jvmti.html)-based method samplers suffer from. This results in stack walking will being safer when used together with [ZGC](https://wiki.openjdk.org/display/zgc/Main), and it makes the method sampler more scalable since stack walking can now happen in multiple threads simultaneously. A new experimental event is added, called SafepointLatency, which records the time it takes for a method to reach a safepoint. It’s disabled by default, but you can enable it like this:

    $ java -XX:StartFlightRecording:jdk.SafepointLatency#enabled=true,filename=s.jfr …
    $ jfr print --events jdk.SafepointLatency s.jfr

The second, [JEP 509: JFR CPU-Time Profiling (Experimental)](https://openjdk.org/jeps/509), contains an experimental Linux-only event that uses [SIGPROF](https://www.gnu.org/software/libc/manual/html_node/Alarm-Signals.html) to record method samples. The current method sampling event, jdk.ExecutionSample, works on all platforms, but it only samples events running Java methods. The new jdk.CPUTimeSample also takes into account methods executing in native code, for example, a call to a native method using the new [FFM API](https://openjdk.org/jeps/454) add in JDK 22. The feature builds on the JEP 518: JFR Cooperative Sampling to ensure that the stack can be walked safely.

The third, [JEP 520: JFR Method Timing & Tracing](https://openjdk.org/jeps/520), adds two new events to trace and time methods. Timing and tracing method invocations can help identify performance bottlenecks, optimize code, and find the root causes of bugs. The JEP text demonstrates several command-line examples, so they will not be repeated here. However, I created a simple GUI to validate the design and demonstrate how tools can leverage the events.

![Method Tracer GUI]({{ site.baseurl }}/assets/method-tracer-ui.png){: class="center_85" }

The source code is available on GitHub, and you can run the application like this:

    $ git clone https://github.com/flight-recorder/method-tracer 
    $ java method-tracer/MethodTracer.java

The application is Swing-based and can connect to a local or remote application over [JMX](https://docs.oracle.com/en/java/javase/24/jmx/introduction-jmx-technology.html). It uses [JFR Event Streaming](https://openjdk.org/jeps/349) and [Remote Recording Streaming](https://egahlin.github.io/2021/05/17/remote-recording-stream.html) for data transfer. You can try it out when [early-access builds](https://jdk.java.net/25/) of the JEP are available (expected in build 26). If you find issues, please report them to the [hotspot-jfr-dev](https://mail.openjdk.org/mailman/listinfo/hotspot-jfr-dev) mailing list or send a direct message to [@ErikGahlin](https://x.com/ErikGahlin).

The [‘jfr’ command](https://docs.oracle.com/en/java/javase/24/docs/specs/man/jfr.html) has been updated. If you use ‘jfr scrub’, the tool now prints if an event was removed. This validates that sensitive information is no longer part of the recording.

    $ jfr scrub --exclude-events jdk.InitialSystemProperty,jdk.InitialEnvironmentVariable s.jfr
    Removed events:
    jdk.InitialEnvironmentVariable 23/23
    jdk.InitialSystemProperty      15/15

The ‘jfr print’ command adds a new option “--exact” that prints timestamp, timespan, and memory data with full precision. For example:

    $ jfr print --exact recording.jfr
    jdk.JavaMonitorWait {
      startTime = 20:30:33.091317500 (2025-05-31)
      duration = 0.033899709 s
      monitorClass = sun.java2d.metal.MTLRenderQueue$QueueFlusher (classLoader = bootstrap)
     notifier = "AWT-EventQueue-0" (javaThreadId = 32)
     timeout = 0.100000000 s
     timedOut = false
     address = 0x600000334820
     eventThread = "Java2D Queue Flusher" (javaThreadId = 35)
     stackTrace = [
      java.lang.Object.wait0(long)
      java.lang.Object.wait(long) line: 389
      sun.java2d.metal.MTLRenderQueue$QueueFlusher.run() line: 206
      java.lang.Thread.run() line: 1447
      ...
     ]
    }

This can be useful if you want to compare exact values with a previous run or need exact information in bug reports. For more information, see the [CSR](https://bugs.openjdk.org/browse/JDK-8354195)

The -XX:StartFlightRecording command gets a new option called report-on-exit. This can be used to print a report/view when the JVM exits. For more information about views, see this [blog post](https://egahlin.github.io/2023/05/30/views.html). In the following example, the new method timing event is used to print the time it took for class initializer to execute. This is useful if you want to find places where code could be improved to reduce startup.

    $ java '-XX:StartFlightRecording:method-timing=::<clinit>,report-on-exit=method-timing' -jar J2Ddemo.jar
   
                                            Method Timing
    
    Timed Method                                                                 Invocations Average Time
    ---------------------------------------------------------------------------- ----------- ------------
    java.awt.GraphicsEnvironment$LocalGE.<clinit>()                                        1 39.200000 ms
    sun.font.HBShaper.<clinit>()                                                           1 32.400000 ms
    java2d.DemoFonts.<clinit>()                                                            1 21.400000 ms
    java.nio.file.TempFileHelper.<clinit>()                                                1 16.200000 ms
    java.awt.Component.<clinit>()                                                          1 14.300000 ms
    sun.font.SunFontManager.<clinit>()                                                     1 10.900000 ms
    sun.java2d.SurfaceData.<clinit>()                                                      1  9.480000 ms
    java.awt.Toolkit.<clinit>()                                                            1  8.500000 ms
    java.awt.Font.<clinit>()                                                               1  8.330000 ms
    java.security.Security.<clinit>()                                                      1  7.570000 ms
     ...

Another example of the report-on-exit option is to print a summary of GC pauses when the application exits:

    $ java -XX:StartFlightRecording:report-on-exit=gc-pauses -jar -jar J2Ddemo.jar
   
    GC Pauses
-    --------
    
    Total Pause Time: 331 ms
    
    Number of Pauses: 101
    
    Minimum Pause Time: 0.00629 ms
    
    Median Pause Time: 0.861 ms
    
    Average Pause Time: 3.28 ms
    
    P90 Pause Time: 13.0 ms
    
    P95 Pause Time: 19.7 ms
    
    P99 Pause Time: 21.7 ms
    
    P99.9% Pause Time: 21.7 ms
    
    Maximum Pause Time: 21.7 ms
    

For more information about the feature, see the [CSR](https://bugs.openjdk.org/browse/JDK-8351370) or use the new [-XX:StartFlightRecording:help](https://bugs.openjdk.org/browse/JDK-8326338) command introduced in [JDK 24](https://openjdk.org/projects/jdk/24/).

JDK 25 will add support for [Rate-limited sampling of Java events](https://bugs.openjdk.org/browse/JDK-8351594). For example, you may want to track data that are posted to a queue at a very high frequency. Recording every event can cause the recording to be filled with queue-related events, potentially displacing other important data. By annotating your event with @Throttle, you can set an upper limit of the number of events per seond. For example:
 
    @Throttle(“300/s”) @Label(“Post Message”)
    @Name(“example.PostMessage”)
    @Category(“Message Queue”)
    static class PostMessageEvent extends Event {
      @Label(“Message”)
      String message; 
    }
    
    void postMessage(Channel channel, String message) {
      channel.publish(message);
      PostMessageEvent e = new PostMessageEvent();
      if (e.shouldCommit()) {
        e.message = message.length() < 26 ? message : message.substring(0,22) + “…”);
        e.commit();
      }
    }

Event objects that are throttled cannot be reused. The reason for this is that the event must hold state if it is to be sampled between a call to shouldCommit() and commit(). Like other event settings, the throttling behavior can be controlled from the command line. The following example shows how throttling can be disabled so that all events are emitted:

     $ java -XX:StartFlightRecording:example.PostMessage#throttle=off …
     
JDK 25 also come with a new annotation to help tools visualize the contextual information. Contextual information is data that applies to all events happening in the same thread from the beginning to the end of the event with a field annotated with Contextual.

For example, to trace requests or transactions in a system, a trace event can be created to provide context.

    @Label("Trace")
    @Name("com.example.Trace")
    class TraceEvent extends Event {
      @Label("ID")
      @Contextual
      String id;

      @Label("Name")
      @Contextual
      String name;
    }

To track details within an order service, an order event can be created where only the order ID provides context.

    @Label("Order")
    @Name("com.example.Order")
    class OrderEvent extends Event {
      @Label("Order ID")
      @Contextual
      long id;
    
      @Label("Order Date")
      @Timestamp(Timestamp.MILLISECONDS_SINCE_EPOCH)
      long date;
    
      @Label("Payment Method")
      String paymentMethod;
    }

If an order in the order service stalls due to lock contention, a user interface can display contextual information together with the JavaMonitorEnter event to simplify troubleshooting, for example:

    $ jfr print --events JavaMonitorEnter recording.jfr
    jdk.JavaMonitorEnter {
      Context: Trace.id = "00-0af7651916cd43dd8448eb211c80319c-00f067aa0ba902b7-01"
      Context: Trace.name = "POST /checkout/place-order"
      Context: Order.id = 314159
      startTime = 17:51:29.038 (2025-02-07)
      duration = 50.56 ms
      monitorClass = java.util.ArrayDeque (classLoader = bootstrap)
     previousOwner = "Order Thread" (javaThreadId = 56209, virtual = true)
     address = 0x60000232ECB0
     eventThread = "Order Thread" (javaThreadId = 52613, virtual = true)
     stackTrace = [
       java.util.zip.ZipFile$CleanableResource.getInflater() line: 685
       java.util.zip.ZipFile$ZipFileInflaterInputStream.<init>(ZipFile) line: 388
       java.util.zip.ZipFile.getInputStream(ZipEntry) line: 355
       java.util.jar.JarFile.getInputStream(ZipEntry) line: 833
       ...
     ]
    }

With JDK 24, the Security Manager was [permanently disabled](https://openjdk.org/jeps/486), which enabled the removal of around 3,000 lines of JFR code in JDK 25. You may notice this as faster startup when using JFR, as the number of classes that need to be loaded is reduced. But more importantly, OpenJDK developers no longer need to analyze every new feature to make it work with the Security Manager.

Expect a more rapid stream of enhancements going forward!
