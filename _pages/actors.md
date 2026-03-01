---
layout: default
title: "Actors"
permalink: /actors/
---

# Actor System

Single-threaded actor system that runs on the same event loop as I/O. No locks, no threads, no inter-thread queues.

## Architecture

```
TActorSystem
│
├── Mailbox[0]  actor #1 ──── IActor / ICoroActor / IBehaviorActor
├── Mailbox[1]  actor #2
├── ...
│
├── Timer heap  ─── delayed sends (Schedule / Sleep)
│
└── TNode[nodeId=2] ─── TCP → remote TActorSystem
    TNode[nodeId=3] ─── TCP → remote TActorSystem
```

### TActorId

```
TActorId { NodeId, LocalActorId, Cookie }
           │         │              └── increments on slot reuse
           │         └── index in the actor table
           └── identifies the machine (0 = local)
```

`Cookie` prevents stale references: if an actor dies and its slot is reused, old `TActorId`s silently drop their messages.

---

## Actor Types

```
IActor
  ├── sync Receive(); no co_await; tight dispatch loop
  └── ICoroActor
        └── async CoReceive() → TFuture<void>; one handler at a time

IBehaviorActor
  ├── delegates to a swappable IBehavior*
  └── TBehavior<Derived, Msg1, Msg2, ...>
        ├── typed Receive(MsgN&&, ...) per listed type
        ├── void → sync handler; TFuture<void> → async handler
        └── HandleUnknownMessage() required for unlisted types
```

---

## IActor {#iactor}

Use when the handler is pure computation — no timers, no I/O, no waiting for replies.

```cpp
struct TAdd   { static constexpr TMessageId MessageId = 1; int Value; };
struct TQuery { static constexpr TMessageId MessageId = 2; };
struct TValue { static constexpr TMessageId MessageId = 3; int Result; };

class TCounter : public IActor {
    int Sum = 0;
public:
    void Receive(TMessageId id, TBlob blob, TActorContext::TPtr ctx) override {
        if (id == TAdd::MessageId) {
            Sum += DeserializeNear<TAdd>(blob).Value;
        } else if (id == TQuery::MessageId) {
            ctx->Send<TValue>(ctx->Sender(), Sum);
        }
    }
};
```

`DeserializeNear<T>(blob)` returns `T&` for non-empty POD, `T` by value for empty structs. For messages arriving from a remote node (`blob.Type == Far`) use `DeserializeFar<T>(blob)`, or use `TBehavior` which handles the distinction automatically.

---

## ICoroActor {#icoroactor}

Use when the handler needs to wait — for a timer, for a reply from another actor, or for any async event. Override `CoReceive` instead of `Receive`.

Only one async handler per actor runs at a time. If `CoReceive` suspends, new messages queue in the mailbox and are dispatched after the current handler completes.

```cpp
struct TPing { static constexpr TMessageId MessageId = 10; };
struct TPong { static constexpr TMessageId MessageId = 11; };

class TPingActor : public ICoroActor {
public:
    TFuture<void> CoReceive(TMessageId id, TBlob blob,
                            TActorContext::TPtr ctx) override {
        if (id == TPing::MessageId) {
            co_await ctx->Sleep(std::chrono::milliseconds(100));
            ctx->Send<TPong>(ctx->Sender());
        }
        co_return;
    }
};
```

---

## IBehaviorActor {#ibehavioractor}

Typed dispatch via `TBehavior<Derived, Msg1, Msg2, ...>`. Each message type gets its own `Receive` overload. Handlers can mix sync (`void`) and async (`TFuture<void>`). Call `Become(ptr)` to switch to another behavior object.

```cpp
struct TPing    { static constexpr TMessageId MessageId = 101; };
struct TReply   { static constexpr TMessageId MessageId = 102; std::string Text; };

// Non-POD messages need serialization hooks for cross-node transport:
namespace NNet::NActors {
template<> void SerializeToStream<TReply>(const TReply& r, std::ostringstream& os) {
    os << r.Text;
}
template<> void DeserializeFromStream<TReply>(TReply& r, std::istringstream& is) {
    r.Text = is.str();
}
} // namespace NNet::NActors

class TMyActor : public IBehaviorActor {
    struct TStateA : public TBehavior<TStateA, TPing, TReply> {
        TMyActor* Parent;
        explicit TStateA(TMyActor* p) : Parent(p) {}

        void Receive(TPing&&, TBlob, TActorContext::TPtr ctx) {
            ctx->Send<TReply>(ctx->Sender(), "pong from A");
            Parent->Become(&Parent->StateB);   // switch behavior
        }
        void Receive(TReply&& r, TBlob, TActorContext::TPtr) {
            std::cout << "Got: " << r.Text << "\n";
        }
        void HandleUnknownMessage(TMessageId id, TBlob, TActorContext::TPtr) {
            std::cerr << "Unknown message: " << id << "\n";
        }
    };

    struct TStateB : public TBehavior<TStateB, TPing, TReply> {
        TMyActor* Parent;
        explicit TStateB(TMyActor* p) : Parent(p) {}

        // Async handler — can co_await
        TFuture<void> Receive(TPing&&, TBlob, TActorContext::TPtr ctx) {
            co_await ctx->Sleep(std::chrono::milliseconds(50));
            ctx->Send<TReply>(ctx->Sender(), "pong from B");
            Parent->Become(&Parent->StateA);
            co_return;
        }
        void Receive(TReply&&, TBlob, TActorContext::TPtr) {}
        void HandleUnknownMessage(TMessageId, TBlob, TActorContext::TPtr) {}
    };

public:
    TMyActor() : StateA(this), StateB(this) { Become(&StateA); }
    TStateA StateA;
    TStateB StateB;
};
```

`Become` takes effect immediately — the next message dispatched will use the new behavior.

---

## TActorContext API

| Method | Description |
|---|---|
| `ctx->Self()` | Own `TActorId` |
| `ctx->Sender()` | Sender of the currently processed message |
| `ctx->Send<T>(to, args...)` | Construct and send `T` to `to` |
| `ctx->Forward<T>(to, blob)` | Re-route a message preserving the original sender |
| `ctx->Schedule<T>(when, from, to, args...)` | Delayed send; returns a cancellable handle |
| `ctx->Cancel(handle)` | Cancel a previously scheduled message |
| `co_await ctx->Sleep(duration)` | Suspend for a duration or until a time point |
| `co_await ctx->Ask<TReply>(to, question)` | Request–reply; suspends until reply arrives |

`Ask<TReply>` creates a one-shot internal actor, sends the question, waits for `TReply`, then cleans up. Usable in `ICoroActor` and async `TBehavior` handlers.

---

## Message Types

### POD — automatic serialization

Any type satisfying `std::is_trivially_copyable_v<T> && std::is_standard_layout_v<T>` works out of the box, both locally and across nodes:

```cpp
struct TJob {
    static constexpr TMessageId MessageId = 42;
    int WorkerId;
    float Priority;
};

// Empty structs cost 0 bytes on the wire:
struct TStart { static constexpr TMessageId MessageId = 43; };
```

### Non-POD — explicit serialization

```cpp
struct TTextMsg {
    static constexpr TMessageId MessageId = 44;
    std::string Text;
};

namespace NNet::NActors {
template<> void SerializeToStream<TTextMsg>(const TTextMsg& m, std::ostringstream& os) {
    os << m.Text;
}
template<> void DeserializeFromStream<TTextMsg>(TTextMsg& m, std::istringstream& is) {
    m.Text = is.str();
}
}
```

For distributed use also register with `TMessagesFactory`:

```cpp
TMessagesFactory factory;
factory.RegisterSerializer<TTextMsg>();
```

---

## Local Setup

```cpp
TLoop<TDefaultPoller> loop;
TActorSystem sys(&loop.Poller());

auto counterId = sys.Register(std::make_unique<TCounter>());
sys.Send<TAdd>(TActorId{}, counterId, 5);
sys.Send<TAdd>(TActorId{}, counterId, 3);
sys.Send<TQuery>(TActorId{}, counterId);

while (sys.ActorsSize() > 0)
    loop.Step();
```

---

## Distributed Setup

```
Node 1 (id=1, :8001)                    Node 2 (id=2, :8002)
┌──────────────────────┐                ┌──────────────────────┐
│  TActorSystem        │                │  TActorSystem        │
│  ┌────────────────┐  │                │  ┌────────────────┐  │
│  │ TNode → node 2 │◄─┼────TCP/TLS─────┼─►│ TNode → node 1 │  │
│  └────────────────┘  │                │  └────────────────┘  │
│  actor A             │                │  actor B             │
└──────────────────────┘                └──────────────────────┘

Wire format: [ THeader: Sender | Recipient | MessageId | Size ][ payload bytes ]
```

```cpp
TResolver resolver(loop.Poller());
TMessagesFactory factory;
factory.RegisterSerializer<TMyMsg>();

TActorSystem sys(&loop.Poller(), /*nodeId=*/1);

// Connect to remote node 2
sys.AddNode(2, std::make_unique<TNode<TPoller>>(
    loop.Poller(), factory, resolver,
    [&](const TAddress& addr) {
        return TPoller::TSocket(loop.Poller(), addr.Domain());
    },
    THostPort("node2.example.com", 8002)
));

// Listen for incoming connections
typename TPoller::TSocket listener(loop.Poller(), AF_INET6);
listener.Bind(TAddress{"::", 8001});
listener.Listen();
sys.Serve(std::move(listener));

// Send to remote actor — same API as local
auto remoteId = TActorId{2, actorLocalId, cookie};
sys.Send<TMyMsg>(TActorId{}, remoteId, 42);
```

`TNode` reconnects automatically with 1 s back-off if the TCP connection drops.

---

## Poison Pill

```cpp
ctx->Send<TPoison>(targetId);
```

Terminates the actor after it processes the pill. Its slot is recycled with an incremented `Cookie`, silently invalidating any stale `TActorId`s pointing to it.

---

## Benchmarks

Ring topology: N actors forwarding a message M times (i5-11400F · Ubuntu 25.04).

**Local ring — 100 actors, batch 1024**

| Framework | msg/s |
|---|---|
| Akka | 473,966 |
| **COROIO** | **442,151** |
| CAF | 302,930 |
| YDB/actors | 151,972 |

**Distributed ring — 10 processes, batch 1024, 0-byte payload**

| Framework | msg/s |
|---|---|
| **COROIO** | **1,137,790** |
| YDB/actors | 182,525 |
| CAF | 55,540 |
| Akka | 5,765 |

**Distributed ring — 10 processes, batch 1024, 1024-byte payload**

| Framework | msg/s |
|---|---|
| **COROIO** | **860,188** |
| YDB/actors | 96,372 |

Full source: [ping_actors.cpp](https://github.com/resetius/coroio/blob/master/examples/ping_actors.cpp) · [behavior_actors.cpp](https://github.com/resetius/coroio/blob/master/examples/behavior_actors.cpp)
