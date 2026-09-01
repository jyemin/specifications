# Locality-Aware Mongos Selection

- Status: Proposed
- Minimum Server Version: N/A

______________________________________________________________________

## Abstract

This specification defines how a driver prefers mongos servers that are in the same cloud provider and region as the
application. Locality metadata is published in a single DNS TXT record on a dedicated `_locality` sub-label of the
cluster domain, by the cluster operator. A driver reads that record with one additional DNS query and uses it to order
mongos candidates during server selection.

This specification applies to sharded topologies only. Locality-based selection within a replica set is already
available through read preference tag sets.

## META

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Motivation for Change

A multi-cloud or multi-region sharded cluster spans more than one cloud provider or region. A driver selects a mongos
using the latency window: any mongos whose average RTT is within `localThresholdMS` (default 15 milliseconds) of the
shortest is suitable, and one of those is chosen. When two cloud providers have data centers in the same metro area,
their RTTs commonly fall inside the same window. An application running in AWS is therefore routed through a GCP mongos
for some fraction of its operations, which incurs cross-cloud egress charges and adds latency variance.

The latency window is not a locality control, and three existing mechanisms do not address this:

**Read preference tags** determine which mongod members of a shard serve a read. They have no effect on which mongos the
driver selects.

**`localThresholdMS`** can be lowered to narrow the window, but it selects on latency, not on location. A value low
enough to exclude a remote-cloud mongos in the same metro area also excludes local mongos servers whose RTT is slightly
higher, and the correct value differs between deployments and changes over time.

**`srvMaxHosts`** limits how many hosts from the SRV response populate the seedlist, but selects them randomly. For a
larger sharded cluster, a random sample is likely to contain hosts from remote regions and may contain no local host at
all. A host that is not in the seedlist is never a candidate for selection.

## Specification

### Terms

#### locality map

The driver's in-memory mapping from mongos host to the `provider`, `region`, and `nativeRegion` values parsed from the
`_locality` TXT record.

#### locality tier

One of the three ordered candidate sets defined in [Locality-Aware Server Selection](#locality-aware-server-selection).

### Locality Metadata in DNS

#### Namespace

Locality metadata is published as a single DNS TXT record on the cluster domain prefixed with the `_locality` label:

```text
_locality.<domain>
```

`<domain>` is the hostname from the `mongodb+srv` connection string. For a connection string of
`mongodb+srv://prod-cluster.example.mongodb.net`, the driver queries `_locality.prod-cluster.example.mongodb.net`. The
record is distinct from the TXT record on `<domain>` that carries default connection string options.

The DNS rules for a host name do not permit the underscore character, so an underscore-prefixed label cannot collide
with a mongos host name. Publishing metadata under such a label is an established convention, used by DNS-SD
([RFC 6763](https://www.rfc-editor.org/rfc/rfc6763)), DMARC (`_dmarc.<domain>`), and ACME domain control validation
(`_acme-challenge.<domain>`).

The cluster operator (e.g. Atlas) publishes this record, with a publication latency no greater than that of the SRV and
TXT records it already publishes for `mongodb+srv` URIs.

Exactly one TXT record is published at `_locality.<domain>`. A payload too large for one character string is split
across several character strings within that record. See
[Publishing the Locality Record](#publishing-the-locality-record).

#### Payload Format

The record value is a list of groups separated by `;`. Each group associates one location with the hosts in it:

```text
payload   = group *( ";" group )
group     = provider ":" region ":" nativeRegion ":" hostlist
hostlist  = hostlabel *( "," hostlabel )
```

| Field          | Description                                                                                                                                                  | Example                                   |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------- |
| `provider`     | Cloud provider identifier                                                                                                                                    | `AWS`, `GCP`, `AZURE`                     |
| `region`       | A deployment-specific region identifier. MAY be empty; see below                                                                                             | `US_EAST_1`, `CENTRAL_US`, `EUROPE_NORTH` |
| `nativeRegion` | The cloud provider's own region name, as returned by that provider's instance metadata service (see [Client Locality Detection](#client-locality-detection)) | `us-east-1`, `us-central1`, `northeurope` |
| `hostlabel`    | The leading label or labels of a mongos host name returned by the SRV query, with the parent domain omitted                                                  | `mongos-dc1-01`                           |

For an Atlas cluster, the `provider` and `region` values match the
[pre-defined replica set tags](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/) Atlas sets on co-located
mongod nodes. For a self-hosted cluster, the operator chooses values that suit its configuration, and may leave `region`
empty when a second region name is not needed. See
[Why the payload names each region twice](#why-the-payload-names-each-region-twice).

Example with both region names populated:

```dns
_locality.prod-cluster.example.mongodb.net. 60 IN TXT "AWS:US_EAST_1:us-east-1:mongos1,mongos2;GCP:CENTRAL_US:us-central1:mongos3"
```

This assigns `mongos1.prod-cluster.example.mongodb.net` and `mongos2.prod-cluster.example.mongodb.net` to AWS
`us-east-1`, and `mongos3.prod-cluster.example.mongodb.net` to GCP `us-central1`.

Example with `region` empty, as a self-hosted operator might publish when a second name is not needed:

```dns
_locality.prod-cluster.corp.internal. 60 IN TXT "AWS::us-east-1:mongos-dc1-01,mongos-dc1-02;GCP::us-central1:mongos-dc2-01"
```

#### Payload Size

The payload is larger than a 512-byte DNS message for all but small clusters. For a query on
`_locality.prod-cluster.example.mongodb.net`, the DNS header, the question section, and the answer record's fixed fields
occupy 72 bytes, leaving 440 bytes of RDATA within a 512-byte message. Each host costs its host name label plus one
separator byte, and each group costs approximately 24 bytes:

| Hosts | Label length | Locations | RDATA bytes |
| ----: | -----------: | --------: | ----------: |
|    10 |           24 |         4 |         348 |
|    50 |           24 |         4 |       1,352 |
|   100 |           24 |         4 |       2,607 |

512 bytes is the limit only for DNS messages that do not use EDNS(0)
([RFC 6891](https://www.rfc-editor.org/rfc/rfc6891)). With EDNS(0) a client advertises a larger UDP buffer, commonly
1232 or 4096 bytes, and a response too large for the advertised buffer falls back to TCP.

Drivers MUST NOT assume the response fits in 512 bytes. A truncated or failed response is handled as described in
[Locality Metadata Lookup and Refresh](#locality-metadata-lookup-and-refresh).

#### Parsing the Payload

##### Character string concatenation

The RDATA of a DNS TXT record is one or more `<character-string>`s
([RFC 1035 §3.3.14](https://www.rfc-editor.org/rfc/rfc1035#section-3.3.14)). A `<character-string>` is a single length
octet followed by that many bytes, so it carries at most 255 bytes
([RFC 1035 §3.3](https://www.rfc-editor.org/rfc/rfc1035#section-3.3)). The driver MUST concatenate all character strings
of the record, in the order they are defined, into one string before tokenizing on any delimiter, inserting no separator
between them. The order of character strings within one TXT record is guaranteed.

A group or a host token may be split across a character string boundary:

```dns
_locality.prod-cluster.example.mongodb.net. 60 IN TXT ( "AWS:US_EAST_1:us-east-1:mongos1,mongos2;GCP:CENT"
                                                        "RAL_US:us-central1:mongos3" )
```

`GCP:CENTRAL_US` is split here, and tokenizing each character string separately would corrupt it. This is the same
requirement as for the connection options TXT record in
[Initial DNS Seedlist Discovery](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md#default-connection-string-options).

##### Host matching

A `hostlabel` token matches a mongos host name if the host name, compared case-insensitively, begins with the token
followed immediately by `.`. Tokens are label prefixes rather than fully qualified names, so the parent domain is not
repeated in the payload. Requiring the trailing `.` prevents `mongos1` from matching
`mongos10.prod-cluster.example.mongodb.net`.

##### Multiple records

Exactly one TXT record is valid at `_locality.<domain>`. If the query returns more than one, the driver MUST NOT merge
them and MUST NOT select one of them. It MUST treat the response as a failed lookup, as described in
[Locality Metadata Lookup and Refresh](#locality-metadata-lookup-and-refresh), and SHOULD log the condition.

Merging is not an alternative, because DNS servers may return the records of an RRset in any order. If two records
disagreed about a host, which one took effect would vary between lookups and between clients.

##### Duplicate hosts

A host appears in at most one group. If a host appears in more than one, the driver MUST use the first occurrence in
payload order and ignore the rest. Unlike the multiple-record case, this is well defined, because the order of character
strings within one record is guaranteed.

##### Malformed payloads

The driver MUST NOT raise an error for a malformed payload. The driver MUST skip and continue past:

- a group with fewer than four `:`-separated fields
- a group with more than four `:`-separated fields
- a group with an empty `provider`, `nativeRegion`, or `hostlist`
- an empty group, for example one produced by a trailing or doubled `;`
- an empty host token

An empty `region` field is valid. The driver MUST NOT match `localRegion` against an empty `region`.

The driver MUST ignore host tokens that do not match any host returned by the SRV query. A host returned by the SRV
query that matches no token in the payload has unknown locality and MUST appear only in Tier 3.

All `provider`, `region`, and `nativeRegion` comparisons MUST be case-insensitive, both against the payload and against
the values obtained from [Client Locality Detection](#client-locality-detection).

### MongoClient Options

#### localCloud

The cloud provider the application is running in. Valid values are `"AWS"`, `"GCP"`, `"AZURE"`, `"auto"`, and `null`.
Values are case-insensitive. It MUST be configurable at the client level. It MUST NOT be configurable at the level of a
database object, collection object, or individual operation.

- `"auto"` (default): The driver detects the cloud provider when the MongoClient is constructed, as described in
    [Client Locality Detection](#client-locality-detection). If detection fails, the driver MUST behave as if
    `localCloud` were `null` and MUST NOT raise an error.
- `null`: Locality-aware selection is disabled. The driver behaves as if this specification were not implemented and
    MUST NOT issue the `_locality` TXT query.
- Any other string value: The driver MUST raise a configuration error.

`localCloud` has no effect when `loadBalanced=true`, where there is one endpoint and no mongos selection. The driver
MUST NOT raise an error if both are specified; it MUST ignore `localCloud`.

`localCloud` has no effect on replica set topologies. Selecting replica set members by location is available through
[read preference tag sets](../server-selection/server-selection.md#tag-set-lists).

#### localRegion

The region the application is running in. Valid values are a non-empty string, `"auto"`, and `null`. It MUST be
configurable at the client level. It MUST NOT be configurable at the level of a database object, collection object, or
individual operation.

- `"auto"` (default): The driver detects the region when the MongoClient is constructed, together with the cloud
    provider. If detection fails, `localRegion` is treated as `null`.
- `null`: No region preference is applied. The driver still prefers mongos servers in the same cloud if `localCloud` is
    set.
- A non-empty string: Either the deployment-specific region identifier, for example `US_EAST_1`, or the cloud provider's
    own region name, for example `us-east-1`. The driver MUST accept both, comparing case-insensitively against the
    candidate's `region` field first and its `nativeRegion` field second.

`localRegion` is ignored if `localCloud` is `null` or if cloud detection fails.

Both forms are accepted so that an application which also uses
[read preference tag sets](../server-selection/server-selection.md#tag-set-lists) can write one region value throughout
its configuration, rather than one identifier in the tag set and a different one in `localRegion`:

```text
?readPreferenceTags=provider:AWS,region:US_EAST_1&localCloud=AWS&localRegion=US_EAST_1
```

Detection yields the cloud provider's own region name, because that is what the instance metadata services return. Since
the payload carries both names, the driver resolves either input.

### Connection String Requirements

Locality-aware selection requires the `mongodb+srv` connection string scheme, because the `_locality` record is located
relative to the SRV domain. There is no locality metadata for a seedlist given directly in a `mongodb://` connection
string.

If `localCloud` is set, explicitly or by detection, with a non-SRV connection string, the driver MUST NOT raise an error
and MUST NOT issue the `_locality` query. All hosts have unknown locality, every candidate is in Tier 3, and selection
behaves as it does without this specification. Drivers MUST document this limitation.

### Client Locality Detection

When `localCloud` or `localRegion` is `"auto"`, the driver MUST attempt to detect the cloud provider and region by
querying the following instance metadata endpoints:

| Cloud | Endpoint                                                                                               | Region field                                                                                                            |
| ----- | ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| AWS   | `http://169.254.169.254/latest/meta-data/placement/region`                                             | Response body                                                                                                           |
| GCP   | `http://metadata.google.internal/computeMetadata/v1/instance/zone` (header: `Metadata-Flavor: Google`) | Last path segment of the response body, with the zone suffix removed, for example `us-central1-a` becomes `us-central1` |
| Azure | `http://169.254.169.254/metadata/instance/compute?api-version=2021-02-01` (header: `Metadata: true`)   | The `location` field of the JSON response                                                                               |

Requests MUST use a short timeout, RECOMMENDED 100 milliseconds, and MUST NOT block MongoClient construction if they
fail or time out.

Detection MUST be attempted at most once per MongoClient. Drivers MUST cache the result for the lifetime of the
MongoClient.

### Locality Metadata Lookup and Refresh

The driver SHOULD issue the `_locality` TXT query concurrently with the SRV query.

During [initial seedlist discovery](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md#querying-dns),
the driver issues both queries at the same time and waits for both before populating the seedlist, using the same DNS
timeout it applies to the SRV query. Because the queries overlap, MongoClient construction is not delayed.

During each [rescan](../polling-srv-records-for-mongos-discovery/polling-srv-records-for-mongos-discovery.md), the
driver issues the `_locality` query alongside the SRV query and rebuilds the locality map from the result before adding
or removing hosts. The driver MUST NOT poll the `_locality` record on a separate schedule; the query is driven by
*rescanSRVIntervalMS*.

If the `_locality` query fails, times out, or returns no records, the driver:

- MUST NOT raise an error
- MUST NOT change the topology
- MUST retain the locality map most recently obtained, if any, rather than clearing it
- MUST treat all hosts as having unknown locality if no map has been obtained
- SHOULD log the failure, including the reason if available

Retaining the previous map keeps a transient resolver failure from removing locality preference from a working client.
Locality metadata does not change while a host exists, so a map from a previous rescan is normally still correct. See
[Why failover uses SDAM rather than DNS](#why-failover-uses-sdam-rather-than-dns).

### Locality-Aware Server Selection

When `localCloud` is set, explicitly or by detection, and the topology type is Sharded, the driver MUST filter mongos
candidates by the following tiers before applying `localThresholdMS`:

| Tier | Condition                                                                                          | Notes                                      |
| ---- | -------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| 1    | The candidate's `provider` matches `localCloud` and one of its region fields matches `localRegion` | Same cloud, same region                    |
| 2    | The candidate's `provider` matches `localCloud`                                                    | Same cloud, any region                     |
| 3    | All candidates                                                                                     | Used when Tier 1 and Tier 2 are both empty |

A candidate matches `localRegion` if `localRegion` equals, case-insensitively, either the `region` or the `nativeRegion`
field of the group the candidate is in. See [localRegion](#localregion).

The driver MUST evaluate the tiers in order and use the first non-empty tier as the candidate set for `localThresholdMS`
filtering. The driver MUST NOT use a lower tier if a higher tier contains at least one candidate. When at least one
local mongos is suitable, all operations are therefore routed to a local mongos, whatever the RTT of the remote mongos
servers.

Tier 1 is evaluated only when `localRegion` is not null and the candidate's region fields are known. If `localRegion` is
null, the driver begins at Tier 2.

Tier membership is computed from the servers the existing server selection algorithm has already found suitable, so a
server that is not [Available](../server-selection/server-selection.md#terms) is not a candidate in any tier. The driver
MUST evaluate the tiers on each server selection, against the current TopologyDescription. Failover and recovery
therefore require no DNS change, no connection string change, and no application restart:

- When every local mongos becomes unavailable, SDAM marks them Unknown, Tier 1 and possibly Tier 2 are empty on the next
    selection, and operations use the next non-empty tier.
- When a local mongos becomes Available again, the higher tier is no longer empty and the next selection uses it.

Within the selected tier, `localThresholdMS` filtering and the existing selection rules apply unchanged. This
specification changes which servers are candidates, not how one is chosen from among them.

Drivers MUST store the resolved `provider`, `region`, and `nativeRegion` values in the ServerDescription for the mongos,
or in an equivalent structure consulted during selection, so that selection performs no DNS lookup and no other I/O.
Tier evaluation MUST be done in memory.

A candidate that matches no group in the locality map has unknown locality and MUST appear only in Tier 3. A topology in
which no host has known locality behaves as it does without this specification.

### Interaction with srvMaxHosts

When [srvMaxHosts](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md#srvmaxhosts) is a positive
integer and `localCloud` is set, the driver MUST apply locality preference when choosing which hosts populate the
seedlist, and when adding hosts during
[SRV polling](../polling-srv-records-for-mongos-discovery/polling-srv-records-for-mongos-discovery.md).

The algorithm is:

1. Obtain the locality map from the `_locality` query issued alongside the SRV query.
2. Partition the hosts returned by the SRV query into the tiers defined in
    [Locality-Aware Server Selection](#locality-aware-server-selection).
3. Fill the seedlist up to `srvMaxHosts`, taking hosts from the first non-empty tier and then from lower tiers as
    needed. Within a tier, select randomly, using the same randomization algorithm the driver uses for
    [initial selection](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md#querying-dns).

Partitioning happens before truncation, so a local host is not discarded in favor of a remote one.

If `srvMaxHosts` is zero or is not set, all hosts from the SRV response populate the seedlist and locality preference
applies only during server selection.

See [srvMaxHosts and correlated failure of the seedlist](#srvmaxhosts-and-correlated-failure-of-the-seedlist) in
[Open Questions](#open-questions).

### Log Messages

Refer to the [logging specification](../logging/logging.md) for details on logging implementations in general,
including log levels, log components, and structured versus unstructured logging.

Drivers MUST support logging of locality information via the following log messages. These messages MUST be logged at
`Debug` level and use the `serverSelection` log component.

The types used in the structured message definitions below are demonstrative, and drivers MAY use similar types instead
so long as the information is present.

#### "Locality detection completed" Log Message

This message MUST be logged once per MongoClient, after [Client Locality Detection](#client-locality-detection)
completes. It MUST NOT be logged when neither `localCloud` nor `localRegion` is `"auto"`.

| Key         | Suggested Type | Value                                                       |
| ----------- | -------------- | ----------------------------------------------------------- |
| message     | String         | "Locality detection completed"                              |
| detected    | Boolean        | Whether detection succeeded.                                |
| localCloud  | String         | The detected cloud provider. Omitted when detection failed. |
| localRegion | String         | The detected region. Omitted when detection failed.         |

The unstructured form SHOULD be as follows, using the values defined in the structured format above to fill in
placeholders as appropriate:

> Locality detection completed. Detected: {{detected}}, cloud: {{localCloud}}, region: {{localRegion}}

#### "Locality map updated" Log Message

This message MUST be logged when the locality map changes as the result of a lookup, and when a `_locality` query fails
or returns a response the driver rejects.

| Key        | Suggested Type | Value                                                                                                                                                                                 |
| ---------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| message    | String         | "Locality map updated"                                                                                                                                                                |
| groupCount | Int            | The number of groups in the new locality map.                                                                                                                                         |
| hostCount  | Int            | The number of hosts in the new locality map.                                                                                                                                          |
| failure    | Flexible       | Present only when the lookup failed or the response was rejected. See the [logging specification](../logging/logging.md#representing-errors-in-log-messages) for representing errors. |

The unstructured form SHOULD be as follows, using the values defined in the structured format above to fill in
placeholders as appropriate:

> Locality map updated. Groups: {{groupCount}}, hosts: {{hostCount}}. Failure: {{failure}}

#### "Locality tier changed" Log Message

This message MUST be logged when the locality tier used for a successful server selection differs from the tier used for
the previous successful selection on that MongoClient. Drivers MUST NOT log it for a selection that uses the same tier
as the previous one, since that would produce a message per operation.

| Key          | Suggested Type | Value                                                                                                                                                                                      |
| ------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| message      | String         | "Locality tier changed"                                                                                                                                                                    |
| previousTier | Int            | The tier used for the previous successful selection.                                                                                                                                       |
| newTier      | Int            | The tier used for this selection.                                                                                                                                                          |
| serverHost   | String         | The hostname, IP address, or Unix domain socket path for the selected server.                                                                                                              |
| serverPort   | Int            | The port for the selected server. Optional; not present for Unix domain sockets. When the user does not specify a port and the default (27017) is used, the driver SHOULD include it here. |

The unstructured form SHOULD be as follows, using the values defined in the structured format above to fill in
placeholders as appropriate:

> Locality tier changed from {{previousTier}} to {{newTier}}. Selected server: {{serverHost}}:{{serverPort}}

### Relationship to Other Specifications

This specification does not change the behavior defined in
[Initial DNS Seedlist Discovery](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md) or
[Polling SRV Records for Mongos Discovery](../polling-srv-records-for-mongos-discovery/polling-srv-records-for-mongos-discovery.md).
It defines behavior layered on those specifications, active only when `localCloud` is not null.

This specification does not change read preference behavior, tag set behavior, or `localThresholdMS`. It governs mongos
selection only, and has no effect on which mongod member of a shard serves an operation.

For an Atlas cluster, the `provider` and `region` values in the payload match the
[pre-defined replica set tags](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/) Atlas sets on co-located
mongod nodes, so that an application selecting both its mongos and its reads by region uses one value for each in both
places. For a self-hosted cluster, the operator may align the values with its own replica set tags or choose values that
suit its configuration. This specification does not read replica set tags, and `readPreferenceTags` has no effect on
mongos selection.

A driver that does not implement this specification, and a driver that implements it but cannot resolve the `_locality`
record, behave against a cluster that publishes the record as they do today. The record is additional DNS metadata. It
is not a connection string option and cannot cause an older driver to fail to parse a connection string or to
initialize.

### Publishing the Locality Record

This section collects what a publisher of the `_locality` record provides. It places no requirements on drivers, and the
driver-side handling of a record that departs from any of it is given in [Parsing the Payload](#parsing-the-payload) and
[Locality Metadata Lookup and Refresh](#locality-metadata-lookup-and-refresh).

The cluster operator publishes the record with a latency no greater than that of the SRV and TXT records it already 
publishes for `mongodb+srv` URIs. 

- One TXT record exists at `_locality.<domain>`, in the format given in [Payload Format](#payload-format). A payload too
    large for one character string is split across character strings within that record, not across records.
- For an Atlas cluster, `provider` and `region` match the
    [pre-defined replica set tags](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/) Atlas sets on
    co-located mongod nodes. For a self-hosted cluster, the operator chooses values that suit its configuration, and may
    align them with its own replica set tags. The `region` field MAY be empty when a second region name is not needed.
- `nativeRegion` is the region name the cloud provider's own instance metadata service returns.
- Each `hostlabel` is the leading label or labels of a host name the SRV query returns, with the parent domain omitted.
- A host appears in at most one group.
- The payload contains no whitespace, including around delimiters.
- The DNS infrastructure serving the record serves the resulting response size over both UDP with EDNS(0) and TCP. A
    100-host cluster may exceed a 1232-byte buffer; see [Payload Size](#payload-size).

## Test Plan

To be written.

## Design Rationale

### One TXT record per mongos host name

An earlier design published a TXT record on each mongos host name, for example
`mongos1.example.mongodb.net TXT "provider=AWS region=US_EAST_1 nativeRegion=us-east-1"`.

This requires one DNS query per discovered host on every discovery cycle, which is 50 to 100 additional queries during
MongoClient construction for the cluster sizes this feature targets. Resolving them in sequence adds that many round
trips to construction. Resolving them concurrently requires per-host concurrency during construction, which is not
equally available to all drivers: a driver whose DNS API only blocks needs a thread per outstanding lookup, and a
single-threaded driver has one execution context for all of them. This spec uses one record for the cluster, so the cost
of the lookup does not grow with the number of hosts.

### A parallel SRV record encoding locality in host names

Another design published a second SRV record under a separate label, for example `_locality._mongodb._tcp.<domain>`,
whose targets encoded locality as subdomains such as `aws.us-east-1.mongos1.<domain>`.

This idea is rejected because it puts locality into the host name hierarchy. Mongos hosts are commonly served under
wildcard TLS certificates, for example `*.example.mongodb.net`, and a wildcard matches one label, so
`aws.us-east-1.mongos1.example.mongodb.net` does not match. It also makes locality metadata part of the name used for
SNI and certificate verification, so changing the metadata would change the name.

### Grouping hosts by location

Repeating `provider`, `region`, and `nativeRegion` on every host costs approximately 24 bytes per host that grouping
pays once per location. For a 100-host cluster in four locations with 24-byte host name labels, that is approximately
4,900 bytes of RDATA ungrouped against approximately 2,600 grouped.

Grouping does not bring the record within a 512-byte DNS message, and no arrangement of this data would: identifying 100
hosts requires 100 host name labels, which alone exceed the 440 bytes such a message affords. EDNS(0) and TCP fallback
are what make the record practical. Grouping halves the response at every cluster size and avoids TCP fallback for
mid-sized clusters, and it costs nothing in parsing complexity. See [Payload Size](#payload-size).

### Why the payload names each region twice

A group carries two region names: a deployment-specific identifier in `region`, for example `US_EAST_1`, and the cloud
provider's own name in `nativeRegion`, for example `us-east-1`. The `region` field is optional; a self-hosted operator
who does not need a second name leaves it empty. When it is populated, carrying both names avoids requiring a table
mapping one to the other, in the driver or in the application.

The cloud provider's name is the one the instance metadata services return, so it is required for `localRegion="auto"`
without such a table in the driver.

The deployment-specific identifier is the one an application writes in its own configuration. For an Atlas cluster, this
is the Atlas region identifier, which Atlas also sets in the
[pre-defined replica set tags](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/), so an application
selecting region-local reads writes `readPreferenceTags=provider:AWS,region:US_EAST_1`. For a self-hosted cluster, it is
whatever the operator chose when publishing the record. If the payload carried only the cloud provider's name, an
application that also uses read preference tag sets would have to write one region two ways in one connection string:

```text
?readPreferenceTags=provider:AWS,region:US_EAST_1&localCloud=AWS&localRegion=us-east-1
```

The two names are not related by any rule a user can apply, for example Azure's `EUROPE_NORTH` and `northeurope`, so
using the wrong one would produce an empty Tier 1 and no locality preference, with no error.

A self-hosted operator who does not use read preference tags and does not need a second name leaves `region` empty. The
cost of the field when populated is approximately 12 bytes per group rather than per host, and the number of groups is
the number of locations in the cluster, typically between two and six.

### Why multiple TXT records degrade rather than raise an error

[Initial DNS Seedlist Discovery](../initial-dns-seedlist-discovery/initial-dns-seedlist-discovery.md#default-connection-string-options)
requires a client to raise an error when it finds multiple TXT records on `<domain>`. This spec agrees that multiple
records are invalid and differs on the consequence.

That record carries the `authSource`, `replicaSet`, and `loadBalanced` connection string options. Using the wrong value
for those means authenticating against the wrong database or misidentifying the topology, so raising an error is
correct.

This spec raises an error for a value the application supplied and a developer can correct, which is an invalid
`localCloud` value. For anything that reaches the driver from DNS it degrades to Tier 3 instead.

### Case-insensitive comparison

The payload is written by the cluster operator, `localCloud` and `localRegion` are written by the application, and
detected values come from three cloud metadata services. Requiring an exact case match would let a configuration fail
for a reason not visible in the connection string. DNS host name comparison is already case-insensitive.

## Backwards Compatibility

This specification adds two MongoClient options and one DNS record. It changes the behavior of an existing deployment
only when `localCloud` resolves to a non-null value and the cluster publishes a `_locality` record.

The default value of `localCloud` is `"auto"`, so a driver implementing this specification will attempt cloud detection
where earlier versions did not. On a host that is not in a supported cloud, detection fails and the driver behaves as
before.

## Open Questions

### srvMaxHosts and correlated failure of the seedlist

When `srvMaxHosts` is set, a topology whose hosts are all Unknown is not repopulated. A host that is unreachable but
still present in the SRV response is neither removed nor replaced by
[SRV polling](../polling-srv-records-for-mongos-discovery/polling-srv-records-for-mongos-discovery.md), and the topology
is already at `srvMaxHosts` hosts, so there is no room to add another. This behavior predates this specification: with
`srvMaxHosts=3` against a 100-mongos cluster, a client whose three hosts become unreachable while remaining in DNS has
no path to the other 97 today, with no locality preference involved.

What this specification changes is how likely those hosts are to fail together. Random selection distributes them across
regions and availability zones, so one correlated fault rarely claims all of them. Partitioning by tier places them all
in one region by construction, which is the purpose of the feature, and in doing so removes the fault isolation that
random selection provided incidentally. With `srvMaxHosts=3` and four local mongos servers, the seedlist holds three
local hosts and no remote host.

The severity depends on the failure:

- A full outage of the region the application runs in takes the application down as well, so no mongos selection is
    happening. This case is self-limiting.
- A failure confined to the cluster's mongos in that region leaves a healthy application unable to reach any mongos.
    Loss of a private endpoint, a maintenance event, or resource exhaustion on those hosts all produce this, and all of
    them correlate across hosts that share a region. This is the case worth designing for.
- A single availability zone failure strands the client only if every host in the seedlist was in that zone, which
    becomes likely as `srvMaxHosts` approaches 1.

An application tier deployed in more than one region fails over at the application tier rather than in the driver: each
region's application instances use that region's mongos servers, and a regional failure moves traffic to another
region's instances. A single-region application tier is the shape that depends on driver fallthrough, and it is also the
shape that the first case above takes down anyway.

Candidate resolutions:

- Reserve one seedlist entry for a host outside the first non-empty tier whenever a lower tier is non-empty. The
    reserved host receives no operations while any local mongos is suitable, since the higher tier still wins selection,
    so its cost is one monitoring connection. An operator who set `srvMaxHosts=N` for N hosts of local fan-out gets N-1,
    and `srvMaxHosts=1` leaves no entry to reserve.
- Allow SRV polling to replace a host that has been Unknown for longer than some interval with a host from a lower tier.
    This addresses the pre-existing behavior as well, and requires a change to the polling specification.
- Specify no change, and document that `srvMaxHosts` should exceed the number of local mongos servers for applications
    that rely on the driver rather than the application tier for regional failover.

### Zone granularity

This design stops at the region. On some cloud providers, traffic between zones in one region also incurs egress
charges. Atlas's [pre-defined replica set tags](https://www.mongodb.com/docs/atlas/reference/replica-set-tags/) include
an `availabilityZone` tag, but its format differs by provider, for example `us-east-1a` on AWS, `us-east1-b` on GCP, and
`1` on Azure, where the value is meaningless without the region. Should this specification include availability zones?

### The `_locality` label

The label `_locality` is not registered in the IANA "Underscored and Globally Scoped DNS Node Names" registry
established by [RFC 8552](https://www.rfc-editor.org/rfc/rfc8552). A vendor-scoped alternative such as
`_mongodb-locality` would avoid the registry question entirely.

The practical consequence of the unscoped label appears to be limited. The driver queries a fixed name,
`_locality.<domain>`, and the domain owner — Atlas for `*.mongodb.net`, the operator for a self-hosted zone — controls
every record under it. A collision would require another protocol to define `_locality` for a different purpose and a
cluster operator to run both that protocol and MongoDB at the same parent domain, which is unlikely for Atlas clusters
and within the operator's control for self-hosted ones. The tolerant parsing rules in
[Parsing the Payload](#parsing-the-payload) contain a response from an unrelated record by yielding an empty locality
map.

Is the registry question worth resolving, or is the current label adequate given that the publisher and the consumer of
the record are coordinated?

## Changelog

- 2026-09-01: Replace the per-host-name TXT records with one `_locality.<domain>` TXT record whose payload groups hosts
    by location. Reframe around a self-hosted cluster operator as the primary publisher, with Atlas as a specific case.
    Make the `region` field optional. Specify parsing of character strings, host labels, multiple records, duplicate
    hosts, and malformed payloads. Specify the lookup as concurrent with the SRV query and refreshed by
    *rescanSRVIntervalMS*. Accept either region name in `localRegion`. Add payload size, connection string requirements,
    log messages, and design rationale.

- 2026-04-01: Initial proposal.
