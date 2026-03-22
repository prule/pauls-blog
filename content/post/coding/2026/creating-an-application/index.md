---
layout:     post
title:      "Creating and releasing an application"
description: "Creating an application that makes use of Assetto Corsa Competizione (ACC) telemetry"
date:    2026-03-22
author: "Paul"
publishDate: 2026-03-22
tags:
  - code
  - app
  - kotlin
  - gradle
  - acc
  - simracing
categories: [ code ]
---

## Creating and releasing an application

I love sim racing, so I thought it would be an interesting exercise to create an application that makes use of Assetto Corsa Competizione (ACC) telemetry.

The basic operation is this:

- The client sends a registration message to ACC
- ACC replies and starts sending telemetry messages

This is all done over UDP.

## ACC messages

The first step was to create a library that could parse the message byte streams. There is a little bit of documentation in the way of some C# code and a `ServerAdminHandbook.pdf` that can be found in the ACC installation.

I first looked at [ProtoBuf](https://protobuf.dev/overview/), since it can generate classes from a schema, but it wasn't suitable because it has its own wire protocol. I needed something compatible with how ACC sends its messages, so I settled on [Kaitai Struct](https://kaitai.io):

> Kaitai Struct is a declarative language used to describe various binary data structures, laid out in files or in memory: i.e. binary file formats, network stream packet formats, etc.
>
> The main idea is that a particular format is described in Kaitai Struct language (.ksy file) and then can be compiled with ksc into source files in one of the supported programming languages. These modules will include a generated code for a parser that can read the described data structure from a file or stream and give access to it in a nice, easy-to-comprehend API.

This lets me use my [ksy](https://github.com/prule/acc-messages/tree/main/ksy) files to generate classes representing the various messages sent by ACC. I can receive UDP packets, take the packet byte stream and parse them into classes using the Kaitai generated classes.

Here's an example showing a byte array being parsed into a [BroadcastingEvent](https://github.com/prule/acc-messages/blob/32c9e10a461571214f1d9dfb9664928537ac6aad/src/main/java/io/github/prule/acc/messages/AccBroadcastingInbound.java#L285) of type LAPCOMPLETED:

```java
  @Test
  void example() {
    String hexString = "0705090030313a33332e36393511cf0d0000000000";
    byte[] data = hexStringToByteArray(hexString);
    
    ByteBufferKaitaiStream stream = new ByteBufferKaitaiStream(data);
    AccBroadcastingInbound packet = new AccBroadcastingInbound(stream);

    assertEquals(AccBroadcastingInbound.InboundMsgType.BROADCASTING_EVENT, packet.msgType());
    assertTrue(packet.body() instanceof AccBroadcastingInbound.BroadcastingEvent);

    AccBroadcastingInbound.BroadcastingEvent result = (AccBroadcastingInbound.BroadcastingEvent) packet.body();
    assertEquals(AccBroadcastingInbound.BroadcastType.LAPCOMPLETED, result.type());
    assertEquals("01:33.695", result.msg().data());
    assertEquals(904977, result.timeMs());
    assertEquals(0, result.carId());
  }
```

> There's one issue with this solution - it allows me to READ byte arrays into objects, but it doesn't support WRITING.

To handle writing objects I added `AccBroadcastingClient` - this is a simple class that constructs byte arrays for the 4 messages that could be sent by a client.

Wrapping this up into a library gives me a simple way to read and write the required messages - this is [acc-messages](https://github.com/prule/acc-messages).

## ACC Client

Now I needed a client to handle the UDP socket process and provide a way for an application to register listeners so it can receive the messages it needs for its functionality.

This client depends on `acc-messages` and handles the registration handshake and listens for UDP packets from ACC.

Basic usage would look something like this:

```kotlin
    AccClient(
        AccClientConfiguration(name = "Test", port = 9000, serverIp = "127.0.0.1", connectionPassword = "asd")
    ).connect(
        listOf(
            LoggingListener(),
            CsvWriterListener(java.nio.file.Path.of("./recordings"),),
            RegistrationResultListener()
        )
    )
```

Now if I start ACC, enter a practice session, and run the client I should start receiving messages from ACC. In the configuration shown above, it has listeners:

- `LoggingListener` logs every event to the console
- `CsvWriterListener` writes every event to a CSV file
- `RegistrationResultListener` sends an `Entry list` and `Track data` message to ACC when it receives the registration response. This needs some work but it's really just there so I could make sure to get as many different message types as possible for development and testing.

> Here's the good bit - now that I can record real events sent from ACC, I can create a *simulator* to pretend to be ACC:
> 
> - listen on a UDP port, 
> - wait for the registration message, 
> - then play back the recorded events. 
> 
> From this point I can continue developing without having to run ACC itself, and instead just use the simulator.

Running the simulator looks something like this:

```kotlin
    AccSimulator(
        AccSimulatorConfiguration(
            port = 9000,
            connectionPassword = "asd",
            playbackEventsFile = ClasspathSource("io/github/prule/acc/client/simulator/playback-events.csv"),
        ),
    ).start()
```

You can see what the output from a sample run with the client and simulator looks like here: [Sample Run.md](https://github.com/prule/acc-client/blob/main/docs/Sample%20Run.md).

This work resulted in a second library, [acc-client](https://github.com/prule/acc-client).

## A sample application using the client

Now that the client is operational, I needed a way to demonstrate how an application might use it in a simple "Hello World" fashion.
This could be used as a template for starting other applications.

[acc-client-example](https://github.com/prule/acc-client-example) shows this in action, and serves another demonstration purpose - building native binaries for different platforms (more on this later).

## Putting it all together

So far, "it works on my machine". To get it all to build nicely on GitHub, the output from each library needs to be available. One way would be to publish in a maven repository, but the simplest way is to enable [JitPack](https://jitpack.io).

> JitPack is a package repository and build service for JVM (Java, Kotlin) and Android projects that allows developers to publish libraries directly from a Git repository (like GitHub, GitLab, or Bitbucket). It eliminates the need for manual artifact publishing by building projects on-demand.

How it works:

- **Add Repository**: Add the JitPack repository to your Gradle or Maven build file in the libary you want to be available.
- **Add Dependency**: Add the dependency to other projects that need your library using `com.github.Username:Repo:Tag`.
- **Build**: JitPack builds your library when it is needed, and your project uses the resulting artifact.

Here's how the jar is made available:

- **Trigger**: You push code to GitHub and create a Release or a Tag.
- **On-Demand Build**: When someone (a developer or a build tool) first requests that specific version from the jitpack.io repository, JitPack's servers download your source code.
- **Building**: JitPack runs the build commands (like `./gradlew install` or `mvn install`) on its own infrastructure to create the JAR or AAR file.
- **Serving**: Once the build is finished, JitPack stores the resulting artifact and serves it to the user. Future requests for that same version use the cached file.

By enabling `acc-messages` and `acc-client` this way, dependencies can be found and `acc-client-example` can build and release. 

- https://jitpack.io/#prule/acc-messages
- https://jitpack.io/#prule/acc-client

I'm also using GraalVM to do native builds for macOS, Windows and Linux - see [Native Build Tools](https://graalvm.github.io/native-build-tools/latest/index.html).

A GitHub [workflow](https://github.com/prule/acc-client-example/blob/main/.github/workflows/release-native.yml) triggers when a tag is pushed to `acc-client-example` and the native images are built and available under [releases](https://github.com/prule/acc-client-example/releases).

----

Let me know what you think. If I've made any mistakes or the documentation isn't good enough please let me know. Since this is early days, more needs to be done with these libraries to make them complete, and I'll figure that out as I go.

The relevant repositories have more details on their own parts:

- [acc-messages](https://github.com/prule/acc-messages)
- [acc-client](https://github.com/prule/acc-client)
- [acc-client-example](https://github.com/prule/acc-client-example)

With the infrastructure in place, now I can start on my actual application...!!
