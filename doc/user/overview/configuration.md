---
title: "Configuring Materialize"
description: "Understand the command line arguments that control Materialize's behavior"
menu:
  main:
    parent: 'overview'
    weight: 2
---

All of `materialized`'s command line options can be viewed by running
`materialized --help`. This page lists additional considerations and
best practices when running Materialize.

## Command line flags

### Data directory

Upon startup `materialized` creates a directory where it persists
metadata. By default, this directory is called `mzdata` and is
situated in the current working directory of the materialized
process. Currently, only metadata is persisted in `mzdata`. You can
specify a different directory using the `--data-directory` flag. Upon
start, `materialized` checks for an existing data directory, and will
reinstall source and view definitions from it if one is found.

### Worker threads

A `materialized` instance runs a specified number of timely dataflow
worker threads. By default, `materialized` runs with a single worker
thread. Worker threads can only be specified at startup by setting the
`--threads` flag, and cannot be changed without shutting down
`materialized` and restarting. In the future, dynamically changing the
number of worker threads will be possible over distributed clusters,
see
[#2449](https://github.com/MaterializeInc/materialize/issues/2449).

#### How many worker threads should you run?

Adding worker threads allows Materialize to handle more
throughput. Reducing worker threads consumes fewer resources, and
reduces tail latencies.

In general, you should use the fewest number of worker threads that
can handle your peak throughputs. This is also the most resource
efficient.

You should never run Materialize in a configuration greater than `n-1`
workers, where `n` is the number of *physical* cores. Note that major
cloud providers list the number of hyperthreaded cores (or *virtual*
CPUs). Divide this number by two to get the number of physical cores
available. The reasoning is simple: Timely Dataflow is very
computationally efficient and typically uses all available
computational resources. Under high throuput, you should see each
worker pinning a core at 100% CPU, with no headroom for
hyperthreading. One additional core is required for metadata
management and coordination. Timely workers that have to fight for
physical resources will only block each other.

Example: an `r5d.4xlarge` instance has 16 VCPUs, or 8 physical
cores. The recommended worker setting on this VM is `7`.

### Processes and Addresses

*Warning: You should not attempt running a horizontally-scaled
 Materialize cluster until you have maxed-out vertical-scaling*

`--processes` controls the total number of nodes in a
horizontally-scaled Materialize cluster. The IP addresses of each node
should be specified in a file, one per line, which is specified by the
`--address-file` flag.

When each node is started, it must additionally be told which
`--process` it is, from `0` to `processes - 1`.

Note that multi-node Materialize clusters are **not** supported by
Materialize, and are not permitted under the free usage BSL license
without a separate commercial agreement with Materialize. Moreso, nor
are they efficient! An `x1.32xlarge` instance on AWS has 128 VCPUs,
and will be superior in every way (reliability, cost, ease-of-use) to
a multi-node Materialize cluster with the same total number of
VCPUs. It is our performance goal that Materialize under that
configuration be able to handle every conceivable streaming workload
that you may wish to throw at it.

### Listen Address

By default, `materialized` binds to `0.0.0.0:6875`. This means that
Materialize will accept any incoming SQL connection to port 6875 from
anywhere. It is the responsibility of the network firewall to limit
incoming connections. If you wish to configure `materialized` to only
listen to, e.g. localhost connections, you can set `--listen-addr` to
`localhost:6875`. You can also use this to change the port that
Materialize listens on from the default `6875`.
