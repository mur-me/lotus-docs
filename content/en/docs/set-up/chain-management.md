---
title: "Chain management"
description: "The Lotus chain carries the information necessary to compute the current state of the Filecoin network. This guide explains how to manage several aspects of the chain, including how to decrease your node's sync time by loading the chain from a snapshot."
lead: "The Lotus chain carries the information necessary to compute the current state of the Filecoin network. This guide explains how to manage several aspects of the chain, including how to decrease your node's sync time by loading the chain from a snapshot."
draft: false
menu:
    docs:
        parent: "node-set-up"
weight: 290
toc: true
---

## Syncing

Lotus will automatically sync to the latest _chain head_ by fetching the block headers from the current _head_ down to the last synced epoch. The node then retrieves and verifies all the blocks from the last synced epoch to the current head. Once Lotus is synced, it will learn about new blocks as they are mined for every epoch and verify them accordingly. Every epoch might see a variable number of mined blocks.

Filecoin's blockchain grows relatively fast and a full sync will take a long time. Lotus offers a faster way to sync by using trusted state snapshots. There are two types of snapshot available:

| Name                                 | End height    | Message start height        | State start height          |
| ------------------------------------ | ------------- | --------------------------- | --------------------------- |
| [Lightweight](#lightweight-snapshot) | Current block | Current block - 2000 blocks | Current block - 2000 blocks |
| [Full chain](#full-chain-snapshot)   | Current block | Genesis block               | Current block - 2000 blocks |

### Lightweight snapshot

We recommend most users perform the initial node sync from a lightweight snapshot. These snapshots do not contain the full states of the chain and are not suitable for nodes that need to perform queries against historical state information, such as block explorers. However, they are significantly smaller than full chain snapshots and should be sufficient for most use-cases.

{{< alert icon="warning" >}}
These lightweight state snapshots **do not contain any message receipts**. To get message receipts, you need to sync your Lotus node from the genesis block without using any of these snapshots.
{{< /alert >}}

1. Download the most recent lightweight snapshot and its checksum:

    a. For **mainnet**, command always contains the latest snapshot available for mainnet:

    ```shell
    curl -sI https://fil-chain-snapshots-fallback.s3.amazonaws.com/mainnet/minimal_finality_stateroots_latest.car | perl -ne '/x-amz-website-redirect-location:\s(.+)\.car/ && print "$1.sha256sum\n$1.car"' | xargs wget
    ```

    a. For **testnet**, use the [latest calibration network snapshot](https://www.mediafire.com/file/q7tc2bmcc9d09vv/lotus_cali_snapshot_2021_07_14_high_73770.car.tar.xz/file). Testnet snapshots are maintained by Filecoin community voluntarily, and may not be up-to-date. Please double check before using them.

    ```shell
    curl -sI https://www.mediafire.com/file/q7tc2bmcc9d09vv/lotus_cali_snapshot_2021_07_14_high_73770.car.tar.xz/file
    ```
    
1. Check the `sha256sum` of the downloaded snapshot:

    ```shell with-output
    # Replace the filenames for both `.sha256sum` and `.car` files based on the snapshot you downloaded.
    echo "$(cut -c 1-64 minimal_finality_stateroots_517061_2021-02-20_11-00-00.sha256sum) minimal_finality_stateroots_517061_2021-02-20_11-00-00.car" | sha256sum --check
    ```

    This will output something like:

    ```shell
    minimal_finality_stateroots_517061_2021-02-20_11-00-00.car: OK
    ```

1. Start the Lotus daemon using `--import-snapshot`:

    ```shell
    # Replace the filename for the `.car` file based on the snapshot you downloaded.
    lotus daemon --import-snapshot minimal_finality_stateroots_517061_2021-02-20_11-00-00.car
    ```

{{< alert icon="tip" >}}
We strongly recommend that you download and verify the checksum of the snapshot before importing. However, you can skip the `sha256sum` check and use the snapshot URL directly if you prefer:

```shell
lotus daemon --import-snapshot https://fil-chain-snapshots-fallback.s3.amazonaws.com/mainnet/minimal_finality_stateroots_latest.car
```
{{< /alert >}}

### Full chain snapshot

Full chain snapshots contain every block from genesis until the current tipset. You can trustlessly import these complete snapshots by supplying the `--import-chain` option to recalculate the entire state during import:

```shell
lotus daemon --import-chain https://fil-chain-snapshots-fallback.s3.amazonaws.com/mainnet/complete_chain_with_finality_stateroots_latest.car
```

This operation will take multiple days due to the size and complexity of the Filecoin blockchain.

### Checking sync status

There are two ways to check your Lotus daemon's chain synching progress.

#### Sync status

Use `sync status` to output the current state of your local chain:

```shell
lotus sync status
```

This will output something like:

```shell
sync status:
worker 0:
        Base:   [bafy2bzacecnamqgqmifpluoeldx7zzglxcljo6oja4vrmtj7432rphldpdmm2]
        Target: [bafy2bzaceb4b3ionbbxz4uqoehzkjlt4ayta7bneh2bh5xatnwypeuqypebmw bafy2bzaceb2uct4pawanule5bt2ivepcgqls6e6f52lccofvdyfynyfnsa3aa bafy2bzacealylayv2mpgx7wkf54diu6vqmw5yubdgkauii7q2fb7hvwk4343i] (414300)
        Height diff:    414300
        Stage: header sync
        Height: 414300
        Elapsed: 765.267091ms
```

#### Sync wait

Use `sync wait` to output the state of your current chain as an ongoing process:

```shell
lotus sync wait
```

This will output something like:

```shell
Worker: 0; Base: 0; Target: 414300 (diff: 414300)
State: header sync; Current Epoch: 410769; Todo: 3531
Validated 0 messages (0 per second)
...
```

Use `chain getblock` to check when the last synced block was mined:

```shell
date -d @$(./lotus chain getblock $(./lotus chain head) | jq .Timestamp)
```

This will output something like:

```shell
Mon 24 Aug 2020 06:00:00 PM EDT
```

## Creating a snapshot

A full chain CAR-snapshot can be created with `chain export`:

```shell
lotus chain export <filename>
```

To back up a certain number of the most recent state roots, use the `--recent-stateroots` option, along with how many state roots you would like to backup:

```shell
lotus chain export --recent-stateroots=2000 <filename>
```

To create a _pruned_ snapshot and only include blocks directly referenced by the exported state roots, add the `skip-old-msgs` option:

```shell
lotus chain export --recent-stateroots=2000 --skip-old-msgs <filename>
```

## Restoring a custom snapshot

You can restore snapshots by starting the daemon with the `--import-snapshot` option:

```shell
# Without verification
lotus daemon --import-snapshot <filename>

# With verification
lotus daemon --import-chain <filename>
```

If you do not want the daemon to start once the snapshot has finished, add the `--halt-after-import` flag:

```shell
lotus daemon --import-snapshot --halt-after-import <filename>
```

## Compacting the chain data

It is possible to _prune_ the current chain data used by Lotus to reduce the node's disk footprint by resyncing from a minimal snapshot.

1. Export the chain data:

    ```shell
    lotus chain export --recent-stateroots=901 --skip-old-msgs my-snapshot.car
    ```

1. Stop the Lotus daemon:

    ```shell
    lotus daemon stop
    ```

1. Back up the chain data and create a directory  for chain data:

    ```shell
    mv ~/.lotus/datastore/chain ~/.lotus/datastore/chain_backup
    mkdir ~/.lotus/datastore/chain 
    ```

1. Import the chain data:

    ```shell
    lotus daemon --import-snapshot my-snapshot.car --halt-after-import
    ```

1. Start the daemon:

    ```shell
    lotus daemon 
    ```

1. Open another ssh connection or terminal to check sync status :

    ```shell
    lotus sync status 
    lotus sync wait 
    ```

1. That's it!
