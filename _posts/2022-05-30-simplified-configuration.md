---
layout: post
title:  "Improved Ergdpnomic"
author:  "ErikGahlin"
date:   2022-05-30 08:10:24 +0200
image:  "/assets/jfr-configuration-wizard.png"
published: false
tags: [JFR, JDK 17, Ergonomics]
---

JDK 17 was released with several improvements to JFR ergonomics. 

### Configuration wizard

A new configure command was added to the jfr tool.

    $ jfr configure

The command provides an interactive mode that can configure events using options previously only available in the [JMC](https://www.oracle.com/java/technologies/javase/products-jmc8-downloads.html) Recording Wizard.

![JMC Recording Wizard]({{ site.baseurl }}/assets/recording-wizard.png){: class="center_85" }

To start interactive mode, use the --interactive flag:

    $ jfr configure --interactive

![Interactive Mode]({{ site.baseurl }}/assets/jfr-confguration-wizard.png){: class="center_85" }

By default, the configuration is written to a file called custom.jfc. The file can be passed to -XX:StartFlightRecording or [jcmd](https://docs.oracle.com/en/java/javase/17/docs/specs/man/jcmd.html) when starting a recording. 

    $ java -XX:StartFlightRecording:settings=custom.jfc -jar app.jar

    $ jcmd <pid> JFR.start settings=custom.jfc 

It’s also possible to pass options directly to the jfr tool, for example:

    $ jfr configure method-profiling=high gc=high class-loading=true 

Available options depend on the JDK version. To list what is available for a JDK release do:

    $ jfr help configure 

These are the options available in the default configuration (default.jfc) for JDK 17/18.

      gc=<off|normal|detailed|high|all>

      allocation-profiling=<off|low|medium|high|maximum>

      compiler=<off|normal|detailed|all>

      method-profiling=<off|normal|high|max>

      thread-dump=<off|once|60s|10s|1s>

      exceptions=<off|errors|all>

      memory-leaks=<off|types|stack-traces|gc-roots>

      locking-threshold=<timespan>

      file-threshold=<timespan>

      socket-threshold=<timespan>

      class-loading=<true|false>

Another filename that the default can be set with the –output option:

    $ jfr configure allocation-profiling=maximum –output max-allocation.jfc

    $ java -XX:StartFlightRecording:settings=max-allocation.jfc

If settings are changed, the overhead may exceed 1% and the responsiveness of the application suffer. In particular, the memory-leaks=gc-roots option will stop all Java threads and sweep the heap when a recording ends. This could halt the application for seconds.

It’s also possible to change settings of individual events. This can be useful when creating user-defined events to troubleshoot an application specific issue. Don’t be afraid to add events to your application. If the event is not enabled, the implementation will be [empty](https://github.com/openjdk/jdk/blob/master/src/jdk.jfr/share/classes/jdk/jfr/Event.java). The HotSpot [C2 compiler](https://openjdk.java.net/groups/hotspot/docs/HotSpotGlossary.html) is usually able to [eliminate the event](https://youtu.be/xrdLLx6YoDM?t=1456) if the object doesn’t escape the method.

    $ jfr configure +com.company.MyEvent#enabled=true

The plus sign here means that the specified setting will be added to the default set of settings.

If '+' is omitted, the tool will assume an existing setting is to be changed. Since com.company.MyEvent is not included in the JDK, the tool will fail with an error message. This avoids the risk of creating configuration files with misspelled JDK events. 

To list all available events for a JDK release, execute the following command:

    $ jfr metadata

The following commands show socket and method sampling event settings can be configured individually:

    $ jfr configure jdk.SocketRead#enabled=0ms jdk.SocketWrite#threshold=0ms jdk.SocketWrite#stackTrace=true

    $ jfr configure jdk.ExecutionSample#enabled=true jdk.ExecutionSample#period=10ms 

That said, most of the time it’s easier to just use an option:

    $ jfr configure socket-threshold=0ms method-profiling=high

Options are defined in the .jfc file and it’s possible to create user-defined options that can change settings of multiple events, but how to do that is left left out for another blog post. 

The configure command can also merge settings files:

    4 jfr configure --input my.jfc,default.jfc --output combined.jfc

### Configure events from command line

After reading all this, you may wonder why you can’t specify the options and settings directly when using -XX::StartFlightRecording. You can:

    $ java -XX:StartFlightRecording:allocation-profiling=max -jar app.jar

    $ java -XX:StartFlightRecording:+com.company.MyEvent#enabled=true -jar app.jar

If you want to override settings of you own .jfc file you can also do that:

    $ java -XX:StartFlightRecording:settings=my.jfc com.company.MyEvent#enabled=false -jar app.jar

The plus sign is not necessary here as it will change a setting that already exists in my.jfc. To enable a single event, the option settings=none can be set, which means JFR will start from a blank slate.

    $ java -XX:StartFlihtRecording:settings=none,+com.company.MyEvent#enabled=true

    $ java -XX:StartFlihtRecording:settings=none,+jdk.SocketRead#enabled=true,+jdk.SocketRead#threshold=1ms 

### Log events for debugging

JDK 17 also comes with the capability to write events to the JVM [log](https://openjdk.java.net/jeps/158). This is a debug feature and not meant for production use due to the high overhead of formatting the output and printing events while holding a lock.

To print all user-defined events, with a full stack trace, do the following:

    $ java -Xlog:jrf+event=trace -XX:StartFlightRecording ...

To reduce the stack depth to five lines, use -Xlog:jrf+event=debug. For JDK events, use -Xlog:jfr+system+event. This feature best used together -XX:StartFlightRecording:settings=none, for example:

    $ java -XX:StartFlightRecording:settings=none,+com.company.MyEvent#enabled=true ...

Events are flushed to the log once every second by dedicated Java thread.

# &nbsp; {#posts-label}

## Resources

[jfr tiik](https://docs.oracle.com/en/java/javase/17/docs/specs/man/jfr.html)



