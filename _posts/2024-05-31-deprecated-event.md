---
layout: post
title:  "Deprecated Event"
author:  "ErikGahlin"
date:   2024-05-31 21:10:24 +0200
published: true
tags: [JFR, JDK 22, Event]
---

In JDK 22, an event was added to JFR to detect invocations of deprecated methods. The main use case is to determine if a third-party library depends on methods that are going to be removed, for example, methods related to the Security Manager. See [JEP 411: Deprecate the Security Manager for Removal](https://openjdk.org/jeps/411) for further information.

By detecting the use of a deprecated method early in the development process, there is additional time to upgrade, switch to another library, or file a bug with the maintainer of the library. In the future, JDK Mission Control may be extended with a rule to detect deprecated invocations.

To demonstrate how the event works, two classes will be used, both containing invocations to methods deprecated for removal.

    public class API {
      public static void enableLogging(boolean enable) {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
          public Void run() {
            System.setProperty("log", String.valueOf(enable));
            return null;
          }
        });
      }
      public static void runTask(Runnable task) {
        try {
          task.run();
        } catch (ThreadDeath td) {
          System.out.println("Task stopped.");
        }
      }
    }
    
    public class Service {
      public static void log(String message) {
        String shouldLog = System.getProperty("log", "true");
        if (new Boolean("log")) {
          System.out.print(message);
        }
      }
    }

If the above classes are compiled, three warnings are printed:

    $ javac API.java Service.java
    API.java:7: warning: [removal] AccessController in java.security has been deprecated and marked for removal
                    AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    ^
    API.java:18: warning: [removal] ThreadDeath in java.lang has been deprecated and marked for removal
                    } catch (ThreadDeath td) {
                             ^
    Service.java:4: warning: [removal] Boolean(String) in Boolean has been deprecated and marked for removal
                    if (new Boolean("log")) {
                        ^
    3 warnings

These warnings should be fixed, but if the classes are in a library, perhaps compiled before the methods were deprecated, it will not work. Let's put the classes in a jar file, create an Application class that uses them, and run the application with JFR:

  $ jar cf library.jar *.class
  $ rm *.class *.java

   public class Application {
     public static void main(String... args) throws Exception {
		API.enableLogging(true);
		Class.forName("Service").getMethod("log").invoke(null, "Program started.");
    	}
   }

The deprecated event is enabled by default, so no configuration is needed besides starting JFR:

    $ java -XX:StartFlightRecording:filename=recording.jfr -cp library.jar Application.java 

In the above example, a recording file is written when the application exits. The 'jfr view' command can be used to see the invocations.

    $ jfr view deprecated-methods-for-removal recording.jfr
        
                             Deprecated Methods for Removal
    
    Deprecated Method                                             Called from Class
    ------------------------------------------------------------- -----------------
    java.lang.Boolean.<init>(String)                              Service          
    java.security.AccessController.doPrivileged(PrivilegedAction) API      

Notice that the **API::runTask** method is not listed. There are two reasons for that. First, it's never invoked, and JFR purposely only reports methods that are actually called. Second, the method references a deprecated class, but JFR only tracks calls to deprecated methods.

If you want to know all usages of deprecated APIs in a library, the **jdeprscan** tool is a better alternative:

    $ jdeprscan --for-removal library.jar
    Jar file library.jar:
    class API uses deprecated class java/security/AccessController (forRemoval=true)
    class API uses deprecated class java/lang/ThreadDeath (forRemoval=true)
    class Service uses deprecated method java/lang/Boolean::<init>(Ljava/lang/String;)V (forRemoval=true)

The event emitted by JFR contains both the caller and callee, as we can see if we use the **jfr print** command:

    $ jfr print --events jdk.DeprecatedInvocation recording.jfr

    jdk.DeprecatedInvocation {
      startTime = 00:31:21.837 (2024-06-02)
      method = java.lang.Boolean.<init>(String)
      invocationTime = 00:31:21.834 (2024-06-02)
      forRemoval = true
      stackTrace = [
        Service.log(String) line: 4
        ...
      ]
    }

    jdk.DeprecatedInvocation {
      startTime = 00:31:21.837 (2024-06-02)
      method = java.security.AccessController.doPrivileged(PrivilegedAction)
      invocationTime = 00:31:21.834 (2024-06-02)
      forRemoval = true
      stackTrace = [
        API.enableLogging(boolean) line: 7
        ...
      ]
    }

In the initial design of the event, there was no **stackTrace** field, only the fields **method** and **caller**. By putting the caller as the top frame in the stackTrace field, it worked better with existing tools for visualization. It's not possible to get more than one frame for the event.

The reason the caller class and not the caller method is listed in the deprecated-methods-for-removal view is because JFR piggybacks on method resolution inside the JVM. For the interpreter, a check is only made once per caller class. This means only the first call site for a particular class to a specific method generates an event. If the invocation is JIT compiled, all call sites will be reported. This limitation may be lifted in the future.

To record invocations to methods where @Deprecated(forRemoval=false) has been set, the event setting level can be used. Valid values are **off**, **forRemoval** and **all**.

    $ java -XX:StartFlightRecording:jdk.DeprecatedInvocation#level=all,filename=recording.jfr -cp library.jar Application.java 

I hope this blog post has provided a deeper understanding of how the deprecated event works and its limitations. The greatest benefit of the event will likely be realized in systems that already continuously monitor the JVM. Now, they will be able to detect the use of deprecated methods as well.

