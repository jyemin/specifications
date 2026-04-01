# Locality-Aware Mongos Selection

- Status: Proposed
- Minimum Server Version: N/A

______________________________________________________________________

## Abstract

This specification defines a mechanism for MongoDB drivers to prefer mongos servers that are co-located with the
application in the same cloud provider and region. Locality metadata is published in DNS TXT records by Atlas (or by
self-hosted cluster operators using the same convention), allowing drivers to make informed routing decisions without
connecting to mongod nodes or maintaining lookup tables of provider-native region names. This feature applies to
sharded topologies only; replica set topologies already support locality-based routing via replica set tags.

## META

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Motivation

### The Problem

Multi-cloud Atlas sharded clusters span multiple cloud providers. The driver selects mongos routers using a
latency-window algorithm: any router within `localThresholdMS` (default 15ms) of the fastest one is eligible, and the
driver round-robins across all eligible routers. When two cloud providers share a metro area, their round-trip times
frequently fall within this window. An application running in AWS can therefore be routed through a GCP mongos on every
operation, incurring cross-cloud egress charges and latency variance.

### Why Existing Mechanisms Are Insufficient

**Read preference tags** (e.g., `{"provider": "AWS"}`) govern which mongod members serve reads within a replica set
shard. They have no effect on which mongos the driver selects as its first hop.

**`srvMaxHosts`** limits the number of SRV records used to populate the driver's mongos pool, but selects hosts
randomly with no awareness of locality. For clusters with 50–100 mongos servers, random sampling is likely to include
hosts from remote regions, undermining any subsequent affinity logic.

**Application-level workarounds** (custom server selectors that probe co-located mongod nodes for their replica-set
tags) require direct network access to mongod on port 27017, impose per-discovery connection overhead, and require a
lookup table to map provider-native region names to Atlas region identifiers.

### Goals

1. Enable drivers to prefer local-region mongos servers without connecting to mongod.
2. Eliminate the need for application-maintained region name lookup tables.
3. Work correctly in combination with `srvMaxHosts`.
4. Degrade gracefully when locality metadata is absent or the local region has no healthy mongos.

## Specification

### DNS TXT Record Format

Atlas MUST publish locality metadata as DNS TXT records on each mongos hostname. The publication latency for these
records is no greater than that of the existing SRV and TXT records Atlas publishes for `mongodb+srv` URIs. Operators
of self-hosted clusters MAY publish the same records using the same format to enable this feature.

Each TXT record MUST contain the following space-separated key=value pairs:

| Key            | Description                                          | Example            |
| -------------- | ---------------------------------------------------- | ------------------ |
| `provider`     | Cloud provider identifier (uppercase)                | `AWS`, `GCP`, `AZURE` |
| `region`       | Atlas region identifier for the mongos               | `US_EAST_1`        |
| `nativeRegion` | Cloud provider's own region name for the mongos      | `us-east-1`        |

The `provider` and `region` values MUST match the values Atlas injects into replica-set tags on co-located mongod
nodes. This allows interoperability with read preference tag sets without requiring separate translation.

The `nativeRegion` value MUST use the cloud provider's canonical region identifier as returned by that provider's
instance metadata service (see [Client Locality Detection](#client-locality-detection)).

Example TXT records:

```
mongos1.example.mongodb.net.  TXT  "provider=AWS region=US_EAST_1 nativeRegion=us-east-1"
mongos2.example.mongodb.net.  TXT  "provider=GCP region=CENTRAL_US nativeRegion=us-central1"
mongos3.example.mongodb.net.  TXT  "provider=AZURE region=EUROPE_NORTH nativeRegion=northeurope"
```

When multiple TXT records exist for a hostname, the driver MUST use the first record that contains all three required
keys and MUST ignore records that are missing any key.

### MongoClient Options

#### `localCloud`

The cloud provider the application is running in. Valid values are `"AWS"`, `"GCP"`, `"AZURE"`, `"auto"`, and `null`.

- `"auto"` (default): The driver detects the cloud provider at `MongoClient` construction time by querying cloud
  instance metadata endpoints (see [Client Locality Detection](#client-locality-detection)). If detection fails, the
  driver MUST behave as if `localCloud` were `null` and MUST NOT raise an error.
- `null`: Locality-aware selection is disabled. The driver behaves as if this specification were not implemented.
- Any other string value: The driver MUST raise a configuration error.

`localCloud` has no effect in load-balanced mode (`loadBalanced=true`), where a single endpoint is used and no mongos
selection occurs. Drivers MUST NOT raise an error if `localCloud` is set alongside `loadBalanced=true`; they MUST
silently ignore it.

`localCloud` has no effect on replica set topologies. Replica set member selection based on locality is already
supported via [read preference tag sets](../server-selection/server-selection.md#tag-set-lists).

#### `localRegion`

The cloud provider's native region name for the host the application is running in (e.g., `"us-east-1"`,
`"us-central1"`, `"northeurope"`). Valid values are a non-empty string, `"auto"`, and `null`.

- `"auto"` (default): The driver detects the native region at `MongoClient` construction time alongside cloud provider
  detection. If detection fails, `localRegion` is treated as `null`.
- `null`: No region preference is applied. The driver still prefers same-cloud mongos servers if `localCloud` is set.
- A non-empty string: Matched against the `nativeRegion` TXT record key for each mongos candidate.

`localRegion` is ignored if `localCloud` is `null` or if cloud detection fails.

### Client Locality Detection

When `localCloud` or `localRegion` is `"auto"`, the driver MUST attempt to detect the host's cloud provider and region
by querying the following instance metadata endpoints in order, stopping at the first success:

| Cloud  | Endpoint                                                                           | Region field            |
| ------ | ---------------------------------------------------------------------------------- | ----------------------- |
| AWS    | `http://169.254.169.254/latest/meta-data/placement/region`                         | Response body (string)  |
| GCP    | `http://metadata.google.internal/computeMetadata/v1/instance/zone` (header: `Metadata-Flavor: Google`) | Last path segment of response body (e.g., `us-central1-a` → strip trailing `-[a-z]` → `us-central1`) |
| Azure  | `http://169.254.169.254/metadata/instance/compute?api-version=2021-02-01` (header: `Metadata: true`) | `location` field in JSON response |

Requests MUST use a short timeout (RECOMMENDED: 100ms) and MUST NOT block MongoClient construction if they fail or
time out.

Detection MUST be attempted at most once per `MongoClient` instance. Drivers MAY cache the result for the lifetime of
the `MongoClient`.

### Locality-Aware Server Selection

When `localCloud` is set (explicitly or via auto-detection) and a server selection is being performed for a sharded
topology, the driver MUST filter mongos candidates using the following tiered preference before applying
`localThresholdMS`:

| Tier | Condition                                                                      | Notes                                  |
| ---- | ------------------------------------------------------------------------------ | -------------------------------------- |
| 1    | Candidate's `provider` matches `localCloud` AND `nativeRegion` matches `localRegion` | Same cloud, same region                |
| 2    | Candidate's `provider` matches `localCloud`                                    | Same cloud, any region; fallback if Tier 1 is empty |
| 3    | All candidates                                                                 | Availability over affinity; never fail an operation for routing reasons |

The driver MUST evaluate each tier in order and use the highest-populated tier as the candidate set for the subsequent
`localThresholdMS` filtering. The driver MUST NOT use a lower tier if a higher tier contains at least one candidate.

Tier 1 is only evaluated when `localRegion` is non-null and a `nativeRegion` key is present in the candidate's TXT
record. If `localRegion` is null, the driver skips Tier 1 and starts at Tier 2.

Locality metadata for a candidate is derived from its DNS TXT record (see
[DNS TXT Record Format](#dns-txt-record-format)). Drivers MUST store the parsed `provider`, `region`, and
`nativeRegion` values in the server description for the mongos so that they are available during server selection
without additional DNS lookups. TXT records are fetched when a host is first discovered. Whether they should be
periodically re-fetched is an open question (see [Open Questions](#open-questions)).

A candidate with no TXT record, or a TXT record missing the required keys, is treated as having unknown locality and
MUST only appear in Tier 3.

### Interaction with `srvMaxHosts`

When `srvMaxHosts` is a positive integer and `localCloud` is set, the driver MUST apply locality preference when
selecting which hosts to include in the seedlist and when adding hosts during
[SRV polling](../polling-srv-records-for-mongos-discovery/polling-srv-records-for-mongos-discovery.md).

The selection algorithm is:

1. Retrieve TXT records for all hosts returned by the SRV query.
2. Partition hosts into tiers as defined in [Locality-Aware Server Selection](#locality-aware-server-selection).
3. Fill the seedlist up to `srvMaxHosts` by taking hosts from the highest-populated tier first, then lower tiers as
   needed. Within each tier, select randomly (RECOMMENDED: Fisher-Yates shuffle, consistent with the existing
   `srvMaxHosts` algorithm).

If `srvMaxHosts` is zero or not set, all SRV hosts are included in the seedlist and locality preference applies only
at server selection time.

### Relationship to Other Specifications

This specification does not modify the behavior defined in
[Initial DNS Seedlist Discovery](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md) or
[Polling SRV Records for Mongos Discovery](../polling-srv-records-for-mongos-discovery/polling-srv-records-for-mongos-discovery.md).
It defines new behavior that is layered on top of those specifications and is activated only when `localCloud` is
non-null.

This specification also does not modify read preference tag behavior. The `provider` and `region` values in DNS TXT
records are intentionally aligned with Atlas replica-set tags to allow consistent operator configuration, but they are
used solely to guide mongos selection.

## Open Questions

1. **TXT record polling**: The current spec fetches TXT records once at discovery time. However, if a mongos is moved
   to a different cloud region (e.g., during a topology reconfiguration), the cached locality metadata will be stale
   until the driver restarts. Should drivers re-fetch TXT records for each mongos on a schedule, aligned with SRV
   polling? If so, should a stale or missing TXT record cause the server description's locality fields to be cleared,
   or retained until a fresh record is available?

## Changelog

- 2026-04-01: Initial proposal.
