---
title: "Configuration Parameters"
linkTitle: "Configuration"
weight: 3
description: >
    RedisTimeSeries supports multiple module configuration parameters. All of these parameters can only be set at load-time.
---

## Setting configuration parameters on module load

Setting configuration parameters at load-time is done by appending arguments after the `--loadmodule` argument when starting a server from the command line or after the `loadmodule` directive in a Redis config file. For example:

In [redis.conf](/docs/manual/config/):

```sh
loadmodule ./redistimeseries.so [OPT VAL]...
```

From the [Redis CLI](/docs/manual/cli/), using the [MODULE LOAD](/commands/module-load/) command:

```
127.0.0.6379> MODULE LOAD redistimeseries.so [OPT VAL]...
```

From the command line:

```sh
$ redis-server --loadmodule ./redistimeseries.so [OPT VAL]...
```

## RedisTimeSeries configuration parameters

The following table summerizes which configuration parameters can be set at module load-time and which can be set on run-time:

| Configuration Parameter                 | Load-time          | Run-time             |
| :-------                                | :-----             | :-----------         |
| [NUM_THREADS](#num_threads)             | :white_check_mark: | :white_large_square: |
| [COMPACTION_POLICY](#compaction_policy) | :white_check_mark: | :white_large_square: |
| [RETENTION_POLICY](#retention_policy)   | :white_check_mark: | :white_large_square: |
| [DUPLICATE_POLICY](#duplicate_policy)   | :white_check_mark: | :white_large_square: |
| [CHUNK_TYPE](#chunk_type)               | :white_check_mark: | :white_large_square: |

### NUM_THREADS
The maximal number of per-shard threads for cross-key queries when using cluster mode (TS.MRANGE, TS.MGET, and TS.QUERYINDEX). The value must be equal to or greater than 1. Note that increasing this value may either increase or decrease the performance!

#### Default

`3`

#### Example

```
$ redis-server --loadmodule ./redistimeseries.so NUM_THREADS 3
```

### COMPACTION_POLICY

Default compaction rules for newly created key with `TS.ADD`.

Each rule is separated by a semicolon (`;`), the rule consists of multiple fields that are separated by a colon (`:`):

* Aggregation type: One of the following:
  | aggregator | description                                                      |
  | ---------- | ---------------------------------------------------------------- |
  | `avg`      | arithmetic mean of all values                                    |
  | `sum`      | sum of all values                                                |
  | `min`      | minimum value                                                    |
  | `max`      | maximum value                                                    |
  | `range`    | difference between the highest and the lowest value              |
  | `count`    | number of values                                                 |
  | `first`    | the value with the lowest timestamp in the bucket                |
  | `last`     | the value with the highest timestamp in the bucket               |
  | `std.p`    | population standard deviation of the values                      |
  | `std.s`    | sample standard deviation of the values                          |
  | `var.p`    | population variance of the values                                |
  | `var.s`    | sample variance of the values                                    |
  | `twa`      | time-weighted average of all values (since RedisTimeSeries v1.8) |

* Duration of each time bucket - number and the time representation (Example for 1 minute: 1M)

    * m - millisecond
    * M - minute
    * s - seconds
    * d - day

* Retention time - in milliseconds

* (since RedisTimeSeries v1.8):

  Optional: Time bucket alignment - number and the time representation (Example for 1 minute: 1M)

    * m - millisecond
    * M - minute
    * s - seconds
    * d - day

  Assure that there is a bucket that starts at exactly _alignTimestamp_ and align all other buckets accordingly. Units: milliseconds. Default value: 0 (aligned with the epoch). Example: if _bucketDuration_ is 24 hours (24 * 3600 * 1000), setting _alignTimestamp_ to 6 hours after the epoch (6 * 3600 * 1000) will ensure that each bucket’s timeframe is [06:00 .. 06:00).
  
When a compaction policy is defined, compaction rules will be created automatically for newly created time series, and their key would be set to:
  
* Before RedisTimeSeries v1.8:

   _key_dur_agg_ where _key_ is the key of the source time series, _dur_ is the bucket duration, and _agg_ is the aggregator.
     
* Since RedisTimeSeries v1.8:

   _key_dur_agg_aln_ where _key_ is the key of the source time series, _dur_ is the bucket duration, _agg_ is the aggregator, and _aln_ is the alignment timestamp.

Examples:

- `max:1M:1h` - Aggregate using max over one minute and retain the last hour
- `twa:1d:0:360M` - Aggregate daily [06:00 .. 06:00) using time-weighted average; no expiration

#### Default

No compaction rules.

#### Example

```
$ redis-server --loadmodule ./redistimeseries.so COMPACTION_POLICY max:1m:1h;min:10s:5d:10d;last:5M:10ms;avg:2h:10d;avg:3d:100d
```

### RETENTION_POLICY

Maximum age for samples compared to last event time (in milliseconds) per key, this configuration will set
the default retention for newly created keys that do not have a an override.

#### Default

0

#### Example

```
$ redis-server --loadmodule ./redistimeseries.so RETENTION_POLICY 20
```

### DUPLICATE_POLICY

Is policy for handling insertion (`TS.ADD` and `TS.MADD`) of multiple samples with identical timestamps, with one of the following values:

  | policy     | description                                                      |
  | ---------- | ---------------------------------------------------------------- |
  | `BLOCK`    | ignore any newly reported value and reply with an error          |
  | `FIRST`    | ignore any newly reported value                                  |
  | `LAST`     | override with the newly reported value                           |
  | `MIN`      | only override if the value is lower than the existing value      |
  | `MAX`      | only override if the value is higher than the existing value     |
  | `SUM`      | If a previous sample exists, add the new sample to it so that the updated value is equal to (previous + new). If no previous sample exists, set the updated value equal to the new value. |

#### Precedence order
Since the duplication policy can be provided at different levels, the actual precedence of the used policy will be:

1. `TS.ADD`'s `ON_DUPLICATE_policy` optional argument
2. Key-level policy (as set with `TS.CREATE`'s and `TS.ALTER`'s `DUPLICATE_POLICY` optional argument)
3. The `DUPLICATE_POLICY` module configuration parameter
4. The default policy

#### Default

The default policy is `BLOCK`. Both new and pre-existing keys will conform to this default policy.

#### Example

```
$ redis-server --loadmodule ./redistimeseries.so DUPLICATE_POLICY LAST
```

### CHUNK_TYPE

Default chunk type for automatically created keys when [COMPACTION_POLICY](#COMPACTION_POLICY) is configured.

Possible values: `COMPRESSED`, `UNCOMPRESSED`.


#### Default

`COMPRESSED`

#### Example

```
$ redis-server --loadmodule ./redistimeseries.so COMPACTION_POLICY max:1m:1h; CHUNK_TYPE COMPRESSED
```
