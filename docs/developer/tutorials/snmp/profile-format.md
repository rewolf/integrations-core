# Profile Format Reference

## Overview

SNMP profiles are our way of providing out-of-the-box monitoring for certain makes and models of network devices.

An SNMP profile is materialised as a YAML file with the following structure:

```yaml
sysobjectid: <x.y.z...>

# extends:
#   <Optional list of base profiles to extend from...>

metrics:
  # <List of metrics to collect...>

# metric_tags:
#   <List of tags to apply to collected metrics. Required for table metrics, optional otherwise>
```

## Fields

### `sysobjectid`

_(Required)_

The `sysobjectid` field is used to match profiles against devices during device autodiscovery.

It can refer to a fully-defined OID for a specific device make and model:

```yaml
sysobjectid: 1.3.6.1.4.1.232.9.4.10
```

or a wildcard pattern to address multiple device models:

```yaml
sysobjectid: 1.3.6.1.131.12.4.*
```

or a list of fully-defined OID / wildcard patterns:

```yaml
sysobjectid:
  - 1.3.6.1.131.12.4.*
  - 1.3.6.1.4.1.232.9.4.10
```

### `extends`

_(Optional)_

This field can be used to include metrics and metric tags from other so-called _base profiles_. Base profiles can derive from other base profiles to build a hierarchy of reusable profile mixins.

!!! important
    All device profiles should extend from the `_base.yaml` profile, which defines items that should be collected for all devices.

Example:

```yaml
extends:
  - _base.yaml
  - _generic-if.yaml  # Include basic metrics from IF-MIB.
```

### `metrics`

_(Required)_

Entries in the `metrics` field define which metrics will be collected by the profile. They can reference either a single OID (a.k.a _symbol_), or an SNMP table.

#### Symbol metrics

An SNMP symbol is an object with a scalar type (i.e. `Counter32`, `Integer32`, `OctetString`, etc).

In a MIB file, a symbol can be recognized as an `OBJECT-TYPE` node with a scalar `SYNTAX`, placed under an `OBJECT IDENTIFIER` node (which is often the root OID of the MIB):

```asn
EXAMPLE-MIB DEFINITIONS ::= BEGIN
-- ...
example OBJECT IDENTIFIER ::= { mib-2 7 }

exampleSymbol OBJECT-TYPE
    SYNTAX Counter32
    -- ...
    ::= { example 1 }
```

In profiles, symbol metrics can be specified as entries that specify the `MIB` and `symbol` fields:

```yaml
metrics:
  # Example for the above dummy MIB and symbol:
  - MIB: EXAMPLE-MIB
    symbol:
      OID: 1.3.5.1.2.1.7.1
      name: exampleSymbol
  # More realistic examples:
  - MIB: ISILON-MIB
    symbol:
      OID: 1.3.6.1.4.1.12124.1.1.2
      name: clusterHealth
  - MIB: ISILON-MIB
    symbol:
      OID: 1.3.6.1.4.1.12124.1.2.1.1
      name: clusterIfsInBytes
  - MIB: ISILON-MIB
    symbol:
      OID: 1.3.6.1.4.1.12124.1.2.1.3
      name: clusterIfsOutBytes
```

!!! warning
    Symbol metrics from the same `MIB` must still be listed as separate `metrics` entries, as shown above.

    For example, this is _not_ valid syntax:

    ```yaml
    metrics:
      - MIB: ISILON-MIB
        symbol:
          - OID: 1.3.6.1.4.1.12124.1.2.1.1
            name: clusterIfsInBytes
          - OID: 1.3.6.1.4.1.12124.1.2.1.3
            name: clusterIfsOutBytes
    ```

#### Table metrics

An SNMP table is an object that is composed of multiple entries ("rows"), where each entry contains values a set of symbols ("columns").

In a MIB file, tables be recognized by the presence of `SEQUENCE OF`:

```asn
exampleTable OBJECT-TYPE
    SYNTAX   SEQUENCE OF exampleEntry
    -- ...
    ::= { example 10 }

exampleEntry OBJECT-TYPE
   -- ...
   ::= { exampleTable 1 }

exampleColumn1 OBJECT-TYPE
   -- ...
   ::= { exampleEntry 1 }

exampleColumn2 OBJECT-TYPE
   -- ...
   ::= { exampleEntry 2 }

-- ...
```

In profiles, tables can be specified as entries containing the `MIB`, `table` and `symbols` fields:

```yaml
metrics:
  # Example for the dummy table above:
  - MIB: EXAMPLE-MIB
    table:
      # Identification of the table which metrics come from.
      OID: 1.3.6.1.4.1.10
      name: exampleTable
    symbols:
      # List of symbols ('columns') to retrieve.
      # Same format as for a single OID.
      # Each row in the table will emit these metrics.
      - OID: 1.3.6.1.4.1.10.1.1
        name: exampleColumn1
      - OID: 1.3.6.1.4.1.10.1.2
        name: exampleColumn2
      # ...

  # More realistic example:
  - MIB: CISCO-PROCESS-MIB
    table:
      # Each row in this table contains information about a CPU unit of the device.
      OID: 1.3.6.1.4.1.9.9.109.1.1.1
      name: cpmCPUTotalTable
    symbols:
      - OID: 1.3.6.1.4.1.9.9.109.1.1.1.1.12
        name: cpmCPUMemoryUsed
      # ...
```

#### Table metrics tagging

Table metrics require `metric_tags` to identify each row's metric. 
It is possible to add tags to metrics retrieved from a table in three ways:

##### Using a column within the same table

```yaml
metrics:
  - MIB: IF-MIB
    table:
      OID: 1.3.6.1.2.1.2.2
      name: ifTable
    symbols:
      - OID: 1.3.6.1.2.1.2.2.1.14
        name: ifInErrors
      # ...
    metric_tags:
      # Add an 'interface' tag to each metric of each row,
      # whose value is obtained from the 'ifDescr' column of the row.
      # This allows querying metrics by interface, e.g. 'interface:eth0'.
      - tag: interface
        column:
          OID: 1.3.6.1.2.1.2.2.1.2
          name: ifDescr
```

##### Using a column from a different table with identical indexes

```yaml
metrics:
  - MIB: CISCO-IF-EXTENSION-MIB
    forced_type: monotonic_count
    table:
      OID: 1.3.6.1.4.1.9.9.276.1.1.2
      name: cieIfInterfaceTable
    symbols:
      - OID: 1.3.6.1.4.1.9.9.276.1.1.2.1.1
        name: cieIfResetCount
    metric_tags:
      - MIB: IF-MIB
        column:
          OID: 1.3.6.1.2.1.31.1.1.1.1
          name: ifName
        table: ifXTable
        tag: interface
```

##### Using a column from a different table with different indexes

```yaml
metrics:
  - MIB: CPI-UNITY-MIB
    table:
      OID: 1.3.6.1.4.1.30932.1.10.1.3.110
      name: cpiPduBranchTable
    symbols:
      - OID: 1.3.6.1.4.1.30932.1.10.1.3.110.1.3
        name: cpiPduBranchCurrent
    metric_tags:
      - column:
          OID: 1.3.6.1.4.1.30932.1.10.1.2.10.1.3
          name: cpiPduName
        table: cpiPduTable
        index_transform:
          - start: 1
            end: 7
        tag: pdu_name
```

If the external table has different indexes, use `index_transform` to select a subset of the full index. `index_transform` is a list of `start`/`end` ranges to extract from the current table index to match the external table index. `start` and `end` are inclusive.

External table indexes must be a subset of the indexes of the current table, or same indexes in a different order.

!!! example

    In the example above, the index of `cpiPduBranchTable` looks like `1.6.0.36.155.53.3.246`, the first digit is the `cpiPduBranchId` index and the rest is the `cpiPduBranchMac` index. The index of `cpiPduTable` looks like `6.0.36.155.53.3.246` and represents `cpiPduMac` (equivalent to `cpiPduBranchMac`).

    By using the `index_transform` with start 1 and end 7, we extract `6.0.36.155.53.3.246` from `1.6.0.36.155.53.3.246` (`cpiPduBranchTable` full index), and then use it to match `6.0.36.155.53.3.246` (`cpiPduTable` full index).

    `index_transform` can be more complex, the following definition will extract `2.3.5.6.7` from `1.2.3.4.5.6.7`.

    ```yaml
            index_transform:
              - start: 1
                end: 2
              - start: 4
                end: 6
    ```


##### Using an index

Important: "_index_" refers to one digit of the index part of the row OID. Example, if the column OID is `1.2.3.1.2` and the row OID is `1.2.3.1.2.7.8.9`, the full index is `7.8.9`. In this example, when using `index: 1`, we will refer to `7`, `index: 2` will refer to `8`, and so on.

```yaml
metrics:
  - MIB: CISCO-PROCESS-MIB
    table:
      OID: 1.3.6.1.4.1.9.9.109.1.1.1
      name: cpmCPUTotalTable
    symbols:
      - OID: 1.3.6.1.4.1.9.9.109.1.1.1.1.12
        name: cpmCPUMemoryUsed
    metric_tags:
      # This tagging method is more complex, so let's walk through an example...
      #
      # In CISCO-PROCESS-MIB, we can see that entries in the `cpmCPUTotalTable` are indexed by `cpmCPUTotalIndex`,
      # which corresponds to some sort of CPU position for each row in the table:
      #
      #   cpmCPUTotalEntry OBJECT-TYPE
      #      -- ...
      #      INDEX    { cpmCPUTotalIndex }  # <-- See?
      #
      # We want to tag metrics in this table by this CPU position.
      #
      # To do this, we look up the position of this OID in `INDEX`. Here we see it's in 1st position.
      # So we can reference it here using `index: 1`.
      # (If there were two OIDs in `INDEX`, and we wanted to use the one in 2nd position, then we would have used `index: 2`.)
      #
      # NOTE: currently only indexes that refer to a column in the same table are supported.
      - tag: cpu
        index: 1
```

##### Mapping index to tag string value

You can use the following syntax to map indexes to tag string values.
In the example below, the submitted metrics will be `snmp.ipSystemStatsHCInReceives` with tags like `ipversion:ipv6`.

```yaml
metrics:
- MIB: IP-MIB
  table:
    OID: 1.3.6.1.2.1.4.31.1
    name: ipSystemStatsTable
  forced_type: monotonic_count
  symbols:
  - OID: 1.3.6.1.2.1.4.31.1.1.4
    name: ipSystemStatsHCInReceives
  metric_tags:
  - index: 1
    tag: ipversion
    mapping:
      0: unknown
      1: ipv4
      2: ipv6
      3: ipv4z
      4: ipv6z
      16: dns
```

See meaning of index as used here in [Using an index](#using-an-index) section.


##### Tagging tips

!!! note
    General guidelines on [Datadog tagging](https://docs.datadoghq.com/tagging/) also apply to table metric tags.

    In particular, be mindful of the kind of value contained in the columns used a tag sources. E.g. avoid using a `DisplayString` (an arbitrarily long human-readable text description) or unbounded sources (timestamps, IDs...) as tag values.

    Good candidates for tag values include short strings, enums, or integer indexes.

#### Metric type inference

By default, the [Datadog metric type](https://docs.datadoghq.com/developers/metrics/types/?tab=count) of a symbol will be inferred from the SNMP type (i.e. the MIB `SYNTAX`):

| SNMP type             | Inferred metric type |
| --------------------- | -------------------- |
| `Counter32`           | `rate`               |
| `Counter64`           | `rate`               |
| `Gauge32`             | `gauge`              |
| `Integer`             | `gauge`              |
| `Integer32`           | `gauge`              |
| `CounterBasedGauge64` | `gauge`              |
| `Opaque`              | `gauge`              |

SNMP types not listed in this table are submitted as `gauge` by default.

#### Forced metric types

Sometimes the inferred type may not be what you want. Typically, OIDs that represent "total number of X" are defined as `Counter32` in MIBs, but you probably want to submit them `monotonic_count` instead of a `rate`.

For such cases, you can define a `forced_type`. Possible values and their effect are listed below.

| Forced type                | Description                                                  |
| -------------------------- | ------------------------------------------------------------ |
| `gauge`                    | Submit as a gauge.                                           |
| `rate`                     | Submit as a rate.                                            |
| `percent`                  | Multiply by 100 and submit as a rate.                        |
| `monotonic_count`          | Submit as a monotonic count.                                 |
| `monotonic_count_and_rate` | Submit 2 copies of the metric: one as a monotonic count, and one as a rate (suffixed with `.rate`). |
| `flag_stream`              | Submit each flag of a flag stream as individual metric with value `0` or `1`. See [Flag Stream section](#flag-stream). |

This works on both symbol and table metrics:

```yaml
metrics:
  # On a symbol:
  - MIB: TCP-MIB
    forced_type: monotonic_count
    symbol:
      OID: 1.3.6.1.2.1.6.5
      name: tcpActiveOpens
  # On a table:
  - MIB: IP-MIB
    table:
      OID: 1.3.6.1.2.1.4.31.1
      name: ipSystemStatsTable
    forced_type: monotonic_count
    symbols:
    - OID: 1.3.6.1.2.1.4.31.1.1.4
      name: ipSystemStatsHCInReceives
    - OID: 1.3.6.1.2.1.4.31.1.1.6
      name: ipSystemStatsHCInOctets
```

!!! note
    When used on a table metrics entry, `forced_type` is applied to _all_ symbols in the entry.

    So, if a table contains symbols of varying types, you should use multiple `metrics` entries: one for symbols with inferred metric types, and one for each `forced_type`.

    For example:

    ```yaml
    metrics:
      - MIB: F5-BIGIP-LOCAL-MIB
        table:
          OID: 1.3.6.1.4.1.3375.2.2.5.2.3
          name: ltmPoolStatTable
        # No `forced_type` specified => metric types will be inferred.
        symbols:
          - OID: 1.3.6.1.4.1.3375.2.2.5.2.3.1.2
            name: ltmPoolStatServerPktsIn
          - OID: 1.3.6.1.4.1.3375.2.2.5.2.3.1.4
            name: ltmPoolStatServerPktsOut
          # ...

      - MIB: F5-BIGIP-LOCAL-MIB
        table:
          OID: 1.3.6.1.4.1.3375.2.2.5.2.3
          name: ltmPoolStatTable
        forced_type: monotonic_count
        # All these symbols will be submitted as monotonic counts.
        symbols:
          - OID: 1.3.6.1.4.1.3375.2.2.5.2.3.1.7
            name: ltmPoolStatServerTotConns
          - OID: 1.3.6.1.4.1.3375.2.2.5.2.3.1.23
            name: ltmPoolStatConnqServiced
          # ...
    ```

##### Flag stream

When the value is a flag stream like `010101`, you can use `forced_type: flag_stream` to submit each flag as individual metric with value `0` or `1`. Two options are required when using `flag_stream`:

- `options.placement`: position of the flag in the flag stream (1-based indexing, first element is placement 1).
- `options.metric_suffix`: suffix appended to the metric name for a specific flag, usually matching the name of the flag. 

Example:

```yaml
metrics:
  - MIB: PowerNet-MIB
    symbol:
      OID: 1.3.6.1.4.1.318.1.1.1.11.1.1.0
      name: upsBasicStateOutputState
    forced_type: flag_stream
    options:
      placement: 4
      metric_suffix: OnLine
  - MIB: PowerNet-MIB
    symbol:
      OID: 1.3.6.1.4.1.318.1.1.1.11.1.1.0
      name: upsBasicStateOutputState
    forced_type: flag_stream
    options:
      placement: 5
      metric_suffix: ReplaceBattery
```

This example will submit two metrics `snmp.upsBasicStateOutputState.OnLine` and `snmp.upsBasicStateOutputState.ReplaceBattery` with value `0` or `1`.

[Example of flag_stream usage in a profile](https://github.com/DataDog/integrations-core/blob/e64e2d18529c6c106f02435c5fdf2621667c16ad/snmp/datadog_checks/snmp/data/profiles/apc_ups.yaml#L60-L127).

#### Extract value

If the metric value to be submitted is from a OID with string value and needs to be extracted from it, you can use extract value feature.

`extract_value` is a regex pattern with one capture group like `(\d+)C`, where the capture group is `(\d+)`.

Example use cases respective regex patterns:

- stripping the C unit from a temperature value: `(\d+)C`
- stripping the USD unit from a currency value: `USD(\d+)`
- stripping the F unit from a temperature value with spaces between the metric and the unit: `(\d+) *F`

Example:

**Scalar Metric Example:**

```yaml
metrics:
  - MIB: MY-MIB
    symbol:
      OID: 1.2.3.4.5.6.7
      name: temperature
      extract_value: '(\d+)C'
```

**Table Column Metric Example:**

```yaml
metrics:
  - MIB: MY-MIB
    table:
      OID: 1.2.3.4.5.6
      name: myTable
    symbols:
      - OID: 1.2.3.4.5.6.7
        name: temperature
        extract_value: '(\d+)C'
    # ...
```

In the examples above, the OID value is a snmp OctetString value `22C` and we want `22` to be submitted as value for `snmp.temperature`.

### `metric_tags`

_(Optional)_

This field is used to apply tags to all metrics collected by the profile. It has the same meaning than the instance-level config option (see [`conf.yaml.example`](https://github.com/DataDog/integrations-core/blob/master/snmp/datadog_checks/snmp/data/conf.yaml.example)).

Several collection methods are supported, as illustrated below:

```yaml
metric_tags:
  - OID: 1.3.6.1.2.1.1.5.0
    symbol: sysName
    tag: snmp_host
  - # With regular expression matching
    OID: 1.3.6.1.2.1.1.5.0
    symbol: sysName
    match: (.*)-(.*)
    tags:
        device_type: \1
        host: \2
```

### `metadata`

_(Optional)_

This `metadata` section is used to declare where and how metadata should be collected.

General structure:

```yaml
metadata:
  <RESOURCCE>:  # example: device, interface
    fields:
      <FIELD_NAME>: # example: vendor, model, serial_number, etc
        value: "dell"
```

Supported resources and fields can be found here: [payload.go](https://github.com/DataDog/datadog-agent/blob/main/pkg/collector/corechecks/snmp/internal/metadata/payload.go)

#### Value from a static value

```yaml
metadata:
  device:
    fields:
      vendor:
        value: "dell"
```

#### Value from an OID (symbol) value

```yaml
metadata:
  device:
    fields:
      vendor:
        value: "dell"
      serial_number:
        symbol:
          OID: 1.3.6.1.4.1.12124.2.51.1.3.1
          name: chassisSerialNumber
```

#### Value from an OID (symbol) value with regex match

```yaml
metadata:
  device:
    fields:
      vendor:
        value: "dell"
      version:
        symbol:
          OID: 1.3.6.1.2.1.1.1.0
          name: sysDescr
          match_pattern: 'Isilon OneFS v(\S+)'
          match_value: '$1'
          # Will match `8.2.0.0` in `device-name-3 263829375 Isilon OneFS v8.2.0.0`
```

Regex groups captured in `match_pattern` can be used in `match_value`. `$1` is the first captured group, `$2` is the second captured group, and so on.

#### Value from multiple OIDs (symbols)

When the value might be from multiple symbols, we try to get the value from first symbol, if the value can't be fetched (e.g. OID not available from the device), we try to get the value from the second symbol, and so on.

```yaml
metadata:
  device:
    fields:
      vendor:
        value: "dell"
      model:
        symbols:
          - OID: 1.3.6.100.0
            name: someSymbolName
          - OID: 1.3.6.101.0
            name: someSymbolName
```

All OID values are fetched, even if they might not be used in the end. In the example above, both `1.3.6.100.0` and `1.3.6.101.0` are retrieved.
