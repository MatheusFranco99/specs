# Topic Streams Extension

| Lifecycle Stage | Maturity      | Status | Latest Revision |
| --------------- | ------------- | ------ | --------------- |
| 1A              | Working Draft | Active | r0, 2026-06-29  |

Authors: [@marcopolo]

Interest Group: [@sukunrt], `TODO: expand`

[@marcopolo]: https://github.com/marcopolo
[@sukunrt]: https://github.com/sukunrt

See the [lifecycle document][lifecycle-spec] for context about the maturity level
and spec status.

[lifecycle-spec]: https://github.com/libp2p/specs/blob/master/00-framework-01-spec-lifecycle.md

## Overview

Gossipsub v1.3 uses a single stream per direction for all RPCs. This introduces
some problems: unnecessary head of line blocking between messages (especially
problematic when a large message delays small messages) and topic name overhead
on each message.

The Topic Streams Extension addresses these problems. It moves topic scoped
application messages to separate long lived streams.

## Negotiation

Peers MUST use this extension with each other when both sides advertise support
for this extension in the `ControlExtensions` message.

The `ControlExtensions` message is the first message a peer sends and comes at
the same time or earlier than a peer's subscriptions. Since a publisher should
not publish messages to a peer before learning of its subscriptions, there is
no window when a publisher wishes to publish a message, but does not know if
the peer supports Topic Streams.

## Topic Streams

A peer opens a bidirectional stream for each topic that it wishes to send
application messages for. Despite being bidirectional streams, they are treated
as unidirectional streams. If both sides wish to publish messages for a given
topic, both sides must open a bidirectional stream.

It is a protocol violation for the responder of the bidirectional stream to
write anything on the stream. The initiator SHOULD close the connection to the
peer.

The protocol id for a topic stream is `/gsts/v0beta`.

### Control Stream

When this extension is negotiated, the original gossipsub stream becomes the
control stream. Application messages (such as the `Message` or
`PartialMessagesExtension` messages) MUST NOT be published on the control
stream.

### Topic Stream Header

Upon opening the topic stream, the initiator MUST send a single length-prefixed
`TopicRPCHeader` protobuf message. The receiver uses this to identify the topic
associated with the stream. Afterward only length-prefixed `TopicRPC` protobuf
messages are allowed.

If there are multiple streams for a single topic, the receiver SHOULD process
them in the order they were received. The receiver SHOULD limit the number of
concurrent topic streams for the same topic to 3 and downscore peers that open
more. Initiators SHOULD limit the number of concurrent topic streams to 1 per
topic.

If the receiver receives a topic stream for a topic it is not subscribed to, it
SHOULD downscore the peer. If the receiver downscores a peer it MUST NOT
downscore a peer for opening a topic stream that was recently unsubscribed from
as the peer may not have received the unsubscribe message before opening the
topic stream.

### Lifecycle

A topic stream is created when a node publishes a topic message to a peer. It is
closed when either the peer unsubscribes from the topic, or the publisher will
no longer publish on the topic. Either side may close the stream.

Implementations MAY choose to keep topic streams open only for mesh and fanout
peers and use a short lived stream when responding to `IWANTs`.

### Topic Scoped Messages

When a peer wishes to publish a message, it MUST publish a `TopicScopedMessage`
and it MUST NOT publish a message on the control stream.

Implementations MUST NOT set the topic name when sending the message over the
wire.

When verifying a message's signature, implementations MUST reconstruct the full
`Message` by setting `topic` to the value from the stream's `TopicRPCHeader`, and
verify the signature against that. Implementations SHOULD also populate the
`topic` before delivering the message to the application.

### Partial Messages

If the partial message extension has been negotiated with this extension, peers
MUST send each other Partial Messages on the topic stream, not the control
stream.

`PartialMessagesExtension` protobuf messages MUST omit the `topicID` field when
sent over the wire. Implementations SHOULD set the `topicID` field after
reading the message from the wire.
