# Edera Protect

https://edera.dev/

https://github.com/edera-dev

Protect computing workloads with advanced isolation technology.

This repo contains Edera's public protobuf definitions.

These protobufs are published to a [`buf.build` repo here](https://buf.build/edera-dev/protect).

You can include these protos in your own `buf.yaml` like so:

``` yaml
deps:
  - buf.build/edera-dev/protect
```

The folder hierarchy, file naming and package naming schemes follow the [Buf style guide](https://buf.build/docs/best-practices/style-guide/).

## Consuming the API

The Edera control API a Protobuf API exposed locally on each Edera node via a Unix socket.

The Unix socket typically lives at `/var/lib/edera/protect/daemon.socket`.

For a complete code example of a standalone Edera control API client, including connecting to the socket, issuing command RPCs, and subscribing to event streams, [see this Rust crate](https://github.com/edera-dev/falco_plugin/tree/0b14483783d38ddeceb514d63ebf5b0555e8a8e1/src/source_plugin/client).
