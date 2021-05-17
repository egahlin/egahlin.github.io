---
layout: post
title:  "Remote Recording Stream"
author:  "ErikGahlin"
date:   2021-05-17 08:10:24 +0200
image:  "/assets/remote-streaming-architecture.png"
tags: [JFR, JDK 16, Event Streaming]
---

Application monitoring tools have for a long time been able to fetch data continuously over the network using JMX. For example, the CPU load can be obtained from the OperatingSystemMXBean and visualized in JDK Mission Control. JFR provides richer data that is structured, for example stack traces and timestamped values, but until [JDK 16](https://jdk.java.net/16/) there hasn't been a way to transfer this information over the network as it occurs.

In JDK 14, [API support ](https://openjdk.java.net/jeps/349) was added to stream events, illustrated by the following code snippets:

#### 1. Passive, in process:

    try (EventStream stream = EventStream.openRepository()) {
        stream.onEvent("jdk.JavaMonitorEnter", System.out::println),
        stream.start();
    }

#### 2. Passive, out of process:
    
    Path path = Path.of("/repository/2021_05_16_09_48_31_60185");
    try (EventStream stream = EventStream.openRepository(path) {
        stream.onEvent("jdk.JavaMonitorEnter", System.out::println),
        stream.start();
    }

#### 3. Active, in process:

    try (RecordingStream stream = new RecordingStream()) {
        stream.enable("jdk.JavaMonitorEnter").withStackTrace();
        stream.onEvent("jdk.JavaMonitorEnter", System.out::println),
        stream.start();
    }

Active here means that the recording lifecycle and event configuration can be controlled by the stream, as seen in the third code snippet with the enabled method. The above APIs work for many scenarios, but not when there is a need to:

   - monitor a Java process on a remote host.
   - control what is being recorded, on a remote host and/or for another process.

Since JDK 11, there exists a [FlightRecordingMXBean](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.management.jfr/jdk/management/jfr/FlightRecorderMXBean.html) in OpenJDK that can control and download recordings remotely. This is how [JDK Mission Control](https://www.oracle.com/java/technologies/jdk-mission-control.html) fetches recording data and configure events on a remote machine. In JDK 15, and earlier releases, a recording must be stopped before data can be read by a client.

In JDK 16, this restriction was lifted and JFR can now be used to monitor a remote host using a [MBeanServerConnection](https://docs.oracle.com/en/java/javase/16/docs/api/java.management/javax/management/MBeanServerConnection.html).

#### 4. Active, out of process (and over the network)

    String host = "com.example";
    int port = 7091;
 
    String url = "service:jmx:rmi:///jndi/rmi://" + host + ":" + port + "/jmxrmi";
 
    JMXServiceURL u = new JMXServiceURL(url);
    JMXConnector c = JMXConnectorFactory.connect(u);
    MBeanServerConnection connection = c.getMBeanServerConnection();

    try (RemoteRecordingStream stream = new RemoteRecordingStream(connection)) {
        stream.enabled("jdk.JavaMonitorEnter").withStackTrace();
        stream.onEvent("jdk.JavaMonitorEnter", System.out::println),
        stream.start();
    }

The implementation of RemoteRecordingStream reads bytes of data from the [FlightRecorderMXBean::readStream(long)](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.management.jfr/jdk/management/jfr/FlightRecorderMXBean.html#readStream(long)) method and writes it to disk locally, in chunks, similar to what the JVM does on the remote host. Another thread then parses the data on disk and dispatches events to the onEvent handlers. Once every second, new data becomes available to read.

To make sure the parser thread doesn't read data before a data segment is complete, there is a size field in the chunk header that says how far into the file data can be read. Once new data arrive and a segment becomes complete, the field is updated. To make sure the size field is not modified, while being read, there is a protocol the parser must follow to avoid word tearing.

![Remote Streaming Overview]({{ site.baseurl }}/assets/remote-streaming-architecture.png){: class="center_85" }

Once a chunk file has been read and its events dispatched, the file is removed from the client. To instead keep the data, two policies can be set, setMaxAge(Duration) and setMaxSize(long), to determine how long data should be retained.

The RemoteRecordingStream class is not just for streaming events, but also for migrating the disk repository to another host. If the monitored application crashes, and the disk repository files on the host are removed, they are still available on the machine where RemoteRecordingStream operates. Plan is to add a dump method to RemoteRecordingStream, so a file can easily be extracted if something goes wrong.

## Streaming event metadata 

A complication with streaming events from another process is the lack of access to event metadata. When streaming in process, the [FlightRecorder::getEventTypes()](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.jfr/jdk/jfr/FlightRecorder.html#getEventTypes()) method can be invoked to get a list of all registered event types.

Without knowledge of the event types, it's not possible to determine the field layout, or which events that can be enabled/configured.

To remedy the situation, a new method was added to the [EventStream](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.jfr/jdk/jfr/consumer/EventStream.html) interface:

    void onMetadata(MetadataEvent);
    
The [MetadataEvent](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.jfr/jdk/jfr/consumer/MetadataEvent.html) carries a list of all registered event types and the two configurations, "default" and "profile", that comes with the JDK. The MetadataEvent is sent before any onEvent handler is invoked. If a new event type is registered/unregistered, an updated MetadataEvent is sent.

To see the [RemoteRecordingStream](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.management.jfr/jdk/management/jfr/RemoteRecordingStream.html) class in action, there exist a small single-file program called [Health Report.java](https://github.com/flight-recorder/health-report) that subscribes to events and prints them to standard out.

![Health Report]({{ site.baseurl }}/assets/HealthReport.png){: class="center_85" }

Usage: 


    $ java HealthReport.java com.example:7091



# &nbsp; {#posts-label}

## Resources

[JEP 328: Flight Recorder](https://openjdk.java.net/jeps/328)

[JEP 349: JFR Event Streaming](https://openjdk.java.net/jeps/349)

[CSR for JFR: Remote Recording Stream](https://bugs.openjdk.java.net/browse/JDK-8253898)

[Javadoc RemoteRecordingStream](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.management.jfr/jdk/management/jfr/RemoteRecordingStream.html)

[Javadoc MetadataEvent](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.jfr/jdk/jfr/consumer/MetadataEvent.html)

[Health Report](https://github.com/flight-recorder/health-report)



