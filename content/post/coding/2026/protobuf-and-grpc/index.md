---
title: "Remote-controlling a simulator with gRPC and Protobuf"
date: 2026-05-12
tags: [kotlin, grpc, protobuf]
description: "Trying out gRPC instead of REST for typesafe multiplatform client/server communication"
---

# Remote-controlling a simulator with gRPC and Protobuf

I have an Assetto Corsa Competizione (ACC) simulator that replays recorded UDP traffic. It is a useful test fixture, but it had a problem: to start or stop a session you had to be at the same machine, running the JVM process directly. Anything that wanted to drive the simulator as part of a larger pipeline — a Cypress test, a dashboard demo, a teammate on another laptop — had no way in.

I wanted three things:

1. Start a playback session, with per-call overrides for things like the events file and the inter-message delay.
2. Stop the running session, idempotently.
3. Ask "what's running right now?" from anywhere on the network.

This post walks through how I wired that up with Protobuf and gRPC, why I chose them, and what the moving parts look like.

## Why Protobuf and gRPC

Plain REST would have worked. I picked gRPC for four reasons:

- **The contract is the source of truth.** A `.proto` file describes every request, response, and field type. Both sides of the wire are regenerated from it. There is no drift between server and client and no hand-written DTOs.
- **Typed, multi-language clients for free.** I'm on the JVM today, but the same `.proto` will generate Python, Go, or TypeScript stubs the day I need them.
- **Coroutine-friendly stubs.** `grpc-kotlin` produces `suspend` functions out of the box, which matches the rest of the codebase.
- **Optional fields with real "unset" semantics.** Proto3's `optional` keyword lets me tell "caller did not set this" apart from "caller set it to zero", which is exactly what I need for partial config overrides.

## The contract

The entire wire contract lives in a single file, `simulator.proto`:

```proto
syntax = "proto3";

package com.github.prule.acc.client.simulator.grpc;

option java_multiple_files = true;
option java_package = "com.github.prule.acc.client.simulator.grpc";

service Simulator {
  rpc Start(StartRequest) returns (StatusResponse);
  rpc Stop(StopRequest) returns (StatusResponse);
  rpc Status(StatusRequest) returns (StatusResponse);
}

message StartRequest {
  string playback_events_file = 1;

  optional int32  port                = 2;
  optional string connection_password = 3;
  optional int64  delay_ms            = 4;
  optional int32  max_events          = 5;
  optional bool   only_player_events  = 6;
}

message StopRequest {}
message StatusRequest {}

message StatusResponse {
  enum State { STOPPED = 0; RUNNING = 1; }
  State         state   = 1;
  RunningConfig running = 2; // populated when state == RUNNING
}

message RunningConfig {
  string playback_events_file = 1;
  int32  port                 = 2;
  string connection_password  = 3;
  int64  delay_ms             = 4;
  int32  max_events           = 5;
  bool   only_player_events   = 6;
}
```

A few decisions worth calling out:

- **`Start` returns `StatusResponse`, not a separate "start response".** Every RPC ends by reporting the same canonical state, so the client only has to know how to render one shape.
- **`Stop` is idempotent.** Calling it on a stopped server returns `STOPPED` rather than erroring.
- **All `StartRequest` fields except the events file are `optional`.** That gives me partial overrides: the server is configured at boot time with sensible defaults, and any individual `Start` call can override one or two values without having to restate the whole config.
- **`RunningConfig` echoes the effective config back.** The caller sees exactly what the server is doing, not just what they asked for.

## Build wiring

I split the implementation across two Gradle modules:

- `simulator-grpc-server` — owns the `.proto` file, has the service implementation, ships a runnable JVM main.
- `simulator-grpc-client` — depends on the same `.proto` (via a shared `srcDir`), generates the stub, and exposes a small Kotlin wrapper plus a CLI.

The protobuf-gradle plugin handles code generation. Here is the relevant block from `simulator-grpc-server/build.gradle.kts`:

```kotlin
plugins {
  kotlin("jvm")
  id("com.google.protobuf")
  application
}

val grpcVersion        = "1.68.1"
val grpcKotlinVersion  = "1.4.1"
val protobufVersion    = "3.25.5"

dependencies {
  implementation("io.grpc:grpc-stub:$grpcVersion")
  implementation("io.grpc:grpc-protobuf:$grpcVersion")
  implementation("io.grpc:grpc-netty-shaded:$grpcVersion")
  implementation("io.grpc:grpc-kotlin-stub:$grpcKotlinVersion")
  implementation("com.google.protobuf:protobuf-kotlin:$protobufVersion")
  implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.2")
}

protobuf {
  protoc { artifact = "com.google.protobuf:protoc:$protobufVersion" }
  plugins {
    id("grpc")   { artifact = "io.grpc:protoc-gen-grpc-java:$grpcVersion" }
    id("grpckt") { artifact = "io.grpc:protoc-gen-grpc-kotlin:$grpcKotlinVersion:jdk8@jar" }
  }
  generateProtoTasks {
    all().forEach {
      it.plugins { id("grpc"); id("grpckt") }
      it.builtins { id("kotlin") }
    }
  }
}
```

Two things to notice:

1. I register both the `grpc` (Java) and `grpckt` (Kotlin coroutine) generators. The coroutine stubs depend on the Java ones, so I want both.
2. The client module re-uses the server's proto sources directly rather than copying them:

   ```kotlin
   sourceSets {
     main { proto { srcDir("../simulator-grpc-server/src/main/proto") } }
   }
   ```

   That keeps a single canonical `.proto` file. If I ever extract it into its own module I can; for two modules, this is plenty.

## The service implementation

The generated `SimulatorGrpcKt.SimulatorCoroutineImplBase` gives me `suspend` overrides for each RPC. The implementation is small enough to fit on a screen:

```kotlin
class SimulatorService(private val defaults: AccSimulatorConfiguration) :
  SimulatorGrpcKt.SimulatorCoroutineImplBase() {

  private val lock = Any()
  private var current: AccSimulator? = null
  private var currentConfig: AccSimulatorConfiguration? = null

  override suspend fun start(request: StartRequest): StatusResponse {
    require(request.playbackEventsFile.isNotBlank()) { "playback_events_file is required" }
    val config = merge(defaults, request)
    synchronized(lock) {
      stopLocked()                       // Start replaces any running session
      val sim = AccSimulator(config)
      sim.start()
      current = sim
      currentConfig = config
    }
    return status()
  }

  override suspend fun stop(request: StopRequest): StatusResponse {
    synchronized(lock) { stopLocked() }
    return status()
  }

  override suspend fun status(request: StatusRequest): StatusResponse = status()

  private fun merge(defaults: AccSimulatorConfiguration, req: StartRequest) =
    defaults.copy(
      port              = if (req.hasPort())              req.port              else defaults.port,
      connectionPassword= if (req.hasConnectionPassword())req.connectionPassword else defaults.connectionPassword,
      playbackEventsFile= FileSource(req.playbackEventsFile),
      delay             = if (req.hasDelayMs())           req.delayMs           else defaults.delay,
      maxEvents         = if (req.hasMaxEvents())         req.maxEvents         else defaults.maxEvents,
      onlyPlayerEvents  = if (req.hasOnlyPlayerEvents())  req.onlyPlayerEvents  else defaults.onlyPlayerEvents,
    )
}
```

Three patterns to lift out of this:

**Optional fields give you real overrides.** Each `optional` field in proto3 generates a `hasFoo()` companion to the getter. I use it to write `if (req.hasPort()) req.port else defaults.port` — a true three-state read (set / unset / required). Without `optional`, an unset `int32 port = 2` arrives as `0`, which is a perfectly valid port number, so you can't tell "I want port 0" from "I didn't say".

**Lock the mutable state, not the RPC.** The service has exactly one piece of shared state — the current simulator — guarded by an `Any` monitor. `Start` and `Stop` enter the lock just long enough to swap pointers; the `AccSimulator` itself runs on its own threads.

**`Start` is "set to this state", not "start a new one".** The fact that `Start` implicitly stops any prior session means callers don't have to coordinate `Stop` followed by `Start`, and the server is in a known state whether the previous run finished cleanly or not.

## Booting the server

The server is plain `io.grpc.ServerBuilder`:

```kotlin
class SimulatorGrpcServer(
  private val grpcPort: Int,
  private val defaults: AccSimulatorConfiguration,
) {
  private val service = SimulatorService(defaults)
  private val server  = ServerBuilder.forPort(grpcPort).addService(service).build()

  fun run() {
    server.start()
    Runtime.getRuntime().addShutdownHook(Thread {
      service.shutdown()
      server.shutdown()
    })
    server.awaitTermination()
  }
}
```

The shutdown hook is the only non-obvious bit. gRPC's `server.shutdown()` stops accepting new RPCs, but the running simulator is a long-lived UDP loop on its own thread. I have to tell *it* to stop too, which is what `service.shutdown()` does.

## The client wrapper

The generated stub is already perfectly usable, but I wrap it once so callers get a friendly Kotlin API:

```kotlin
class SimulatorGrpcClient internal constructor(
  private val channel: ManagedChannel,
  private val stub: SimulatorGrpcKt.SimulatorCoroutineStub,
) : Closeable {

  suspend fun start(
    playbackEventsFile: String,
    port: Int? = null,
    connectionPassword: String? = null,
    delayMs: Long? = null,
    maxEvents: Int? = null,
    onlyPlayerEvents: Boolean? = null,
  ): StatusResponse {
    val req = StartRequest.newBuilder()
      .setPlaybackEventsFile(playbackEventsFile)
      .apply {
        port?.let               { setPort(it) }
        connectionPassword?.let { setConnectionPassword(it) }
        delayMs?.let            { setDelayMs(it) }
        maxEvents?.let          { setMaxEvents(it) }
        onlyPlayerEvents?.let   { setOnlyPlayerEvents(it) }
      }
      .build()
    return stub.start(req)
  }

  suspend fun stop():   StatusResponse = stub.stop(StopRequest.getDefaultInstance())
  suspend fun status(): StatusResponse = stub.status(StatusRequest.getDefaultInstance())

  override fun close() { channel.shutdown().awaitTermination(5, TimeUnit.SECONDS) }

  companion object {
    fun connect(host: String, port: Int): SimulatorGrpcClient {
      val channel = ManagedChannelBuilder.forAddress(host, port).usePlaintext().build()
      return SimulatorGrpcClient(channel, SimulatorGrpcKt.SimulatorCoroutineStub(channel))
    }
  }
}
```

Two niceties this buys me:

- **Nullable Kotlin params map cleanly onto proto `optional` fields.** A `null` Kotlin parameter never touches the builder, which means the field stays "unset" on the wire, which means the server falls back to its default. This is the bit that makes "override one field, leave the others alone" feel natural at the call site.
- **`Closeable` plus a `connect` factory** lets callers write:

  ```kotlin
  SimulatorGrpcClient.connect("localhost", 50051).use { client ->
    client.start(playbackEventsFile = "/recordings/race.csv", delayMs = 5)
    // ... drive a test ...
    client.stop()
  }
  ```

  No channel lifecycle to think about, no manual `shutdown()` calls scattered through tests.

## When to reach for this

If your problem looks like mine — a long-running JVM process that other code on the network wants to drive — Protobuf and gRPC are a strong default. The combination is particularly worth the setup cost when:

- The contract has more than a handful of operations, or those operations have non-trivial parameters.
- You want typed clients in multiple languages without writing each one yourself.
- You care about distinguishing "unset" from "default value", which is awkward over JSON.
- You expect streaming RPCs in the future. gRPC's bidirectional streaming is the same APIs, with one keyword changed in the `.proto`.

If your contract is one or two endpoints, you're only ever going to call them from `curl`, and you don't want to think about codegen, plain HTTP plus JSON is still the right answer.

For me, the calculus was easy: I already wanted typed clients, I wanted partial overrides, and I'll almost certainly want a streaming "tail the simulator events" RPC next. Defining the contract once in `simulator.proto` and letting the toolchain produce both sides cost me an afternoon and has paid for itself every time I have added a field.



---

*Code is at [github.com/prule/acc-client](https://github.com/prule/acc-client).*
