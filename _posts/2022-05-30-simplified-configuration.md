---
layout: post
title:  "Simplified configuration"
author:  "ErikGahlin"
date:   2022-05-30 08:10:24 +0200
image:  "/assets/jfr-configuration-wizard.png"
published: false
tags: [JFR, JDK 17, Ergonomics]
---

JDK 17 was released with several improvements to JFR ergonomics. A new configure command was added to the jfr-tool.

    $ jfr configure

The tool provides an interactive mode that presents the same options that can be found in the JMC Recording Wizard or Template Manager.

![JMC Recording Wizard](/assets/jfr-configuration-wizard.png)

To start the interactive mode use the --interactive flag.

   $ jfr configure --interactive

![Interactive Mode](assets/recording-wizard.png)

By default, the configuration is written to a file called custom.jfc that can be used when starting JFR from command line. 

   $ java -XX:StartFlightRecording:settings=custom.jfc -jar app.jar

Or using jcmd:

   $ jcmd <pid> JFR.start settings=custom.jfc 

It’s also possible to pass argument directly to the jfr tool, for example:

   $ jfr configure method-sample=high gc=high class-loading=true 

Available options depend on the JDK release. To list what is available for a particular release do:

   $ jfr help configure 

These are the options available in the default configuration for JDK 17:

    Options for default.jfc:

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

You can set another filename than the default with the –output option:

    $ jfr configure allocation-profiling=maxium –output max-allocation.jfr

    $ java -XX:StartFlightRecording:settings=max-allocation.jfc

It’s also possible to change the settings of individual events. This can be useful if you create your own events and want to troubleshoot a particular issue. Don’t be afraid to add your own events. If the event is not enabled, the implementation will empty. See https://github.com/openjdk/jdk/blob/master/src/jdk.jfr/share/classes/jdk/jfr/Event.java . C2 compiler is usually able to eliminate all occurrence of the event if the event object doesn’t escape the method. 

    $ jfr configure +com.company.MyEvent#enabled=true

The plus sign means that the specified setting will be added to the default set of settings. If you omit the ‘+’ sign, the tool will assume you want to change settings in the default configuration (default.jfc) and since the JDK doesn’t contain an event called com.company.MyEvent, it will fail with an error message.  If you want to change the setting of default configuration , the plus sign is not needed, for example:

    $ jfr configure jdk.SocketRead#threshold=0ms jdk.SocketWrite#threshold=0ms

That said, most of the time, it’s easier to just change a predefined option:

    $ jfr configure socket-threshold=0ms

The predefined options are defined in the .jfc file and it’s possible to create your own options where you can change settings of multiple events at the same time.

    $ jfr configure –input my.jfc –output verbode.jfc verbosity=high 

How to create your own options will be left to another blog post. 

After reading all this, you may wonder why you can’t specify the options and settings directly using -XX::StartFlightRecording and jcmd JFR.start. You can, for example:

    $ java -XX:StartFlightRecording:allocation-profiling=maximum -jar app.jar

    $ java -XX:StartFlightRecording:+com.company.MyEvent#enabled=true -jaryapp.jar

If you want to override settings of you own .jfc file you can also do that:

    $ java -XX:StartFlightRecording:settings=my.jfc com.company.MyEvent#enabled=false -jar app.jar

The plus sign is not needed here as you will be changing a setting that already exist. When developing events, or only want to use one specific event, you set can settings=none, which means JFR will start from an blank slate.

    $ java -XX:StartFlihtRecording:settings=none,+com.company.MyEvent#enabled=false

    $ java -XX:StartFlihtRecording:settings=none,+jdk.SocketRead#enabled=true,

JDK 17 also comes with capability to write events to the log. This is a developer features and not mean for production use due to the overhead. If you have your own event called com.company.MyEvent and want to write it to standard out:

    $ java -XX:StartFlightRecording:settings=none,+com.company.MyEvent#enabled=true -jar app.jar

If you want to see want to see what causes a System GC or a deptiomization.

$ java -XX:StartFlightRecording:settings=none,+jdk.SystemGC#enabled=true -jar app.jar
$ java -XX:StartFlightRecording:settings=none,+jdk.Deoptimation#enabled=true -jar app.jar


# &nbsp; {#posts-label}

## Resources

[jfr tiik](https://docs.oracle.com/en/java/javase/17/docs/specs/man/jfr.html)



