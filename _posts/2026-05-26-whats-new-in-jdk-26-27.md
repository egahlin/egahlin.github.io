---
layout: post
title:  "JDK 26-27: What’s New in JFR"
date:   2026-05-26 16:30:24 +0200
author:  "ErikGahlin"
published: true
tags: [JFR, JDK 26, JDK 27]
---

**JDK 25** introduced three JFR-related JEPs: [JEP 518: JFR Cooperative Sampling](https://openjdk.org/jeps/518), [JEP 509: JFR CPU-Time Profiling (Experimental)](https://openjdk.org/jeps/509), and [JEP 520: JFR Method Timing & Tracing](https://openjdk.org/jeps/520). In JDK 26, the focus shifted to maintenance and bug fixes, some of which were also backported to **JDK 25**. Still, a few enhancements were added in **JDK 26**, and a new JEP was introduced in **JDK 27**.

## What’s New in JDK 26

The `jdk.ClassDefine` event now has a `source` field that contains the location from which the class was loaded. This is useful for auditing purposes or for determining from which JAR file a class was loaded.

    jdk.ClassDefine {
      startTime = 01:58:57.947 (2026-05-24)
      definedClass = java2d.demos.Transforms.TransformAnim$DemoControls (classLoader = app)
      definingClassLoader = jdk.internal.loader.ClassLoaders$AppClassLoader (id = 2)
      source = "file:/app/J2Ddemo.jar"
      eventThread = "AWT-EventQueue-0" (javaThreadId = 37)
      stackTrace = [
        java.lang.ClassLoader.defineClass1(ClassLoader, String, byte[], int, int, ProtectionDomain, String)
        java.lang.ClassLoader.defineClass(String, byte[], int, int, ProtectionDomain) line: 974
        java.security.SecureClassLoader.defineClass(String, byte[], int, int, CodeSource) line: 145
        jdk.internal.loader.BuiltinClassLoader.defineClass(String, Resource) line: 776
        jdk.internal.loader.BuiltinClassLoader.findClassOnClassPathOrNull(String) line: 691
        ...
      ]
    }

A new `jdk.FinalFieldMutation` event was also added to help locate code paths where final fields are being modified. For more information about how final fields are becoming truly final, see [JEP 500: Prepare to Make Final Mean Final](https://openjdk.org/jeps/500).

    jdk.FinalFieldMutation {
      startTime = 14:28:04.730 (2026-05-24)
      declaringClass = FinalFieldMutationEventTest$C (classLoader = app)
      fieldName = "value"
      eventThread = "MainThread" (javaThreadId = 24)
      stackTrace = [
        FinalFieldMutationEventTest.testFieldSet() line: 84
        jdk.internal.reflect.DirectMethodHandleAccessor.invoke(Object, Object[]) line: 104
        java.lang.reflect.Method.invoke(Object, Object[]) line: 583
        org.junit.platform.commons.util.ReflectionUtils.invokeMethod(Method, Object, Object[]) line: 786
        org.junit.platform.commons.support.ReflectionSupport.invokeMethod(Method, Object, Object[]) line: 514
        ...
      ]
    }

A new `jdk.StringDeduplication` event was also added to help tune string deduplication heuristics. The event is emitted during string deduplication processing and exposes aggregate statistics for each deduplication cycle. It provides information similar to what you can get with `-Xlog:stringdedup*=debug`.

Here is what the event looks like:

    jdk.StringDeduplication {
      startTime = 14:39:05.117 (2026-05-24)
      duration = 0.402 ms
      inspected = 3334
      known = 2099
      shared = 0
      newStrings = 1235
      newSize = 127.4 kB
      replaced = 27
      deleted = 0
      deduplicated = 1602
      deduplicatedSize = 42.4 kB
      skippedDead = 6
      skippedIncomplete = 0
      skippedShared = 0
      processing = 0.389 ms
      tableResize = 0 s
      tableCleanup = 0 s
    }

## What to Expect from JDK 27

**JDK 27** introduces a new help option: `-XX:FlightRecorderOptions:help`. This option describes all available Flight Recorder options for a particular JDK release and includes example command lines.

There is also a new JEP under development, [JEP 536: JFR In-Process Data Redaction](https://openjdk.org/jeps/536), which can redact sensitive information in three events (`jdk.JVMInformation`, `jdk.InitialEnvironmentVariable`, and `jdk.InitialSystemProperty`) before the data is written to JFR buffers. Since **JDK 21**, a `jfr scrub` command has existed to remove these events if they contain passwords, tokens, or other sensitive information:

    $ jfr scrub JVMInformation,InitialEnvironmentVariable,jdk.InitialSystemProperty recording.jfr

The problem is that many users are unaware of this tool, so we wanted a mechanism that could redact the most common secrets by default. Furthermore, removing an event entirely can also remove information that is vital for troubleshooting an application, such as the GC in use or heap size settings.

Another important aspect is that if you use Event Streaming or connect to the application over JMX, the information may leave the host before ever being written to a file. Redaction therefore happens before the data is written to JFR buffers.

For example, you might start an application with JFR enabled like below:

    $ export ACCESS_TOKEN=SECRET_TOKEN

    $ java -XX:StartFlightRecording:filename=recording.jfr \
        -Xmx2G \
        -Djavax.net.ssl.keyStorePassword=SECRET_PASSWORD \
        -jar application.jar \
        --dbpassword ANOTHER_SECRET_PASSWORD --verbose

If you print the contents of the recording, you will see sensitive information being replaced by `[REDACTED]`:

    $ jfr print \
        --events JVMInformation,InitialSystemProperty,InitialEnvironmentVariable \
        recording.jfr

    jdk.JVMInformation {
      startTime = 17:39:02.196 (2026-02-15)
      jvmVersion = "Java HotSpot(TM) 64-Bit Server VM"
      jvmArguments = "-XX:StartFlightRecording:filename=recording.jfr -Xmx2G
         -Djavax.net.ssl.keyStorePassword=[REDACTED]"
      jvmFlags = "N/A"
      javaArguments = "-jar application.jar [REDACTED] --verbose"
      jvmStartTime = 17:39:02.050 (2026-02-15)
      pid = 43671
    }

    jdk.InitialSystemProperty {
      startTime = 17:39:02.196 (2026-02-15)
      key = "javax.net.ssl.keyStorePassword"
      value = "[REDACTED]"
    }

    jdk.InitialSystemProperty {
      startTime = 17:39:02.196 (2026-02-15)
      key = "sun.java.command"
      value = "-jar application.jar [REDACTED] --verbose"
    }

    jdk.InitialEnvironmentVariable {
      startTime = 17:39:02.244 (2026-02-15)
      key = "ACCESS_TOKEN"
      value = "[REDACTED]"
    }

    ...

If you have tokens, passwords, or other sensitive values that are not matched by the default filters, you can add your own filters using:

    -XX:FlightRecorderOptions:redact-key=+filter1...
    -XX:FlightRecorderOptions:redact-argument=+filter2...

You can view the default filters and learn more about the `redact-key` and `redact-argument` options by using `-XX:FlightRecorderOptions:help`.

Also for **JDK 27**, a new field `hostMemoryUsage` was added to the `jdk.ContainerMemoryUsage` event that describes the amount of physical memory currently allocated in the host system.

## Value Events

In parallel with this work, several JFR issues related to the upcoming [JEP 401: Value Classes and Objects (Preview)](https://openjdk.org/jeps/401) were addressed.

We explored adding support for value events, but decided to postpone that work because the resulting behavior did not consistently match what users would intuitively expect from JFR events. We may revisit the idea later if there is sufficient interest after JEP 401 has been integrated.
