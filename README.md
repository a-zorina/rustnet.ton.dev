# README

This HOWTO contains instructions on how to build and configure a RUST validator node in TON blockchain. The instructions and scripts below were verified on Ubuntu 20.04.

> **Note**: For RustCup, [update](#updating-the-node) your node to the latest version.

# Table of Contents
- [Getting Started](#getting-started)
  - [1. System Requirements](#1-system-requirements)
  - [2. Prerequisites](#2-prerequisites)
    - [2.1 Set the Environment](#21-set-the-environment)
    - [2.2 Install Dependencies](#22-install-dependencies)
  - [3. Deploy RUST Validator Node](#3-deploy-rust-validator-node)
  - [4. Check Node synchronization](#4-check-node-synchronization)
  - [5. Configure validator multisignature wallet](#5-configure-validator-multisignature-wallet)
  - [6. Configure DePool](#6-configure-depool)
- [Stopping, restarting and deleting the RUST Node](#stopping-restarting-and-deleting-the-rust-node)
- [Updating the node](#updating-the-node)
- [Logging](#logging)
  - [During deployment](#during-deployment)
  - [During operation](#during-operation)
- [Troubleshooting](#troubleshooting)
  - [1. Couldn’t connect to Docker daemon at http+docker://localhost](#1-couldnt-connect-to-docker-daemon-at-httpdockerlocalhost)
  - [2. thread 'main' panicked error when checking node synchronization](#2-thread-main-panicked-error-when-checking-node-synchronization)
  - [3. Error executing command when checking node synchronization](#3-error-executing-command-when-checking-node-synchronization)
  - [4. Cannot stop/restart/remove node container](#4-cannot-stoprestartremove-node-container)
  - [5. DePool state not updating](#7-depool-state-not-updating)

# Getting Started

## 1. System Requirements
| Configuration | CPU (threads) | RAM (GiB) | Storage (GiB) | Network (Gbit/s)|
|---|:---|:---|:---|:---|
| Minimum |48|64|1000|1| 

SSD/NVMe disks are obligatory.

## 2. Prerequisites
### 2.1 Set the Environment
Adjust (if needed) `rustnet.ton.dev/scripts/env.sh`:

Set `export DEPOOL_ENABLE=yes` in `env.sh` for a depool validator (an elector request is sent to a depool from a validator multisignature wallet).

Set `export DEPOOL_ENABLE=no` in `env.sh` for a direct staking validator (an elector request is sent from a multisignature wallet directly to the elector).
    
    cd rustnet.ton.dev/scripts/
    . ./env.sh 
    
> Note: Make sure to run the script as `. ./env.sh`, not `./env.sh`

### 2.2 Install Dependencies
`install_deps.sh` script supports Ubuntu OS only.

    ./install_deps.sh 
Install and configure Docker according to the [official documentation](https://docs.docker.com/engine/install/ubuntu/). 

**Note**: Make sure to add your user to the docker group, or run subsequent command as superuser:

    sudo usermod -a -G docker $USER

## 3. Deploy RUST Validator Node
Do this step when the network is launched.
Deploy the node:

    ./deploy.sh 2>&1 | tee ./deploy.log

**Note**: the log generated by this command will be located in the rustnet.ton.dev/scripts/ folder and can be useful for troubleshooting.
  
Wait until the node is synced with the masterchain. Depending on network throughput this step may take significant time (up to several hours).

## 4. Check Node synchronization

Use the following command to check if the node is synced:

    docker exec -it rnode /ton-node/tools/console -C /ton-node/configs/console.json --cmd getstats

Script output example:
```
tonlabs console 0.1.0
COMMIT_ID: 0569a2bbb18c1ce966b1ac21cdf193dbd3c5817b
BUILD_DATE: 2021-01-15 19:11:52 +0000
COMMIT_DATE: 2021-01-15 16:26:58 +0300
GIT_BRANCH: master
{
  "masterchainblocktime": 1610742179,
  "masterchainblocknumber": 24191,
  "timediff": 5,
  "in_current_vset_p34": false,
  "in_next_vset_p36": false
}
```
If the `timediff` parameter equals a few seconds, synchronization is complete.

**Note**: The sync process may not start for up to one hour after node deployment, during which this command may result in error messages. If errors persist for more than an hour after deployment, review deployment log for errors and check the network status.

## 5. Configure validator multisignature wallet

There is a small difference between direct staking and DePool validators on this step:

- For direct staking validator it is necessary to create and deploy a validator [SafeMultisig](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig) wallet in `-1` chain.
- For a DePool validator it is necessary to create and deploy a validator [SafeMultisig](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig) wallet in `0` chain.

You can use [TONOS-CLI](https://github.com/tonlabs/tonos-cli) for this purpose. It should be [configured](https://github.com/tonlabs/tonos-cli) to connect to the main.ton.dev network.

Refer to [this document](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#3-create-wallet) for the detailed wallet creation procedure, or follow the links in the short guide below:

1. All wallet custodians should [create seed phrases and public keys](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#31-create-seed-phrases-and-public-keys-for-all-custodians) for themselves. At least three custodians are recommended for validator wallet, one of which will be used by the validator node. All seed phrases should be kept secret by their owners and securely backed up.
2. The wallet deployer (who may or may not be one of the custodians) should gather the **public** keys from all custodians.
3. The wallet deployer should obtain [SafeMultisig contract code](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#22-download-contract-files) from the repository.
4. The wallet deployer should [generate deployment keys](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#32-generate-deployment-key-pair-file).
5. The wallet deployer should [generate validator wallet](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#33-generate-wallet-address) address: **in -1 chain for direct staking validator or in 0 chain for a DePool validator**.
6. Any user should [send at least 1 token](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#34-send-tokens-to-the-new-address-from-another-wallet) to the generated wallet address to create it in the blockchain.
7. The wallet deployer should [deploy the wallet](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#35-deploy-wallet-set-custodians) contact to the blockchain and set all gathered public keys as its custodians. At this step the number of custodian signatures required to make transactions from the wallet is also set (>=2 recommended for validator wallets). Deploy to  -1 chain for direct staking validator or to 0 chain for a DePool validator.
7. In case of direct staking, the funds for staking should be [transferred](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#46-create-transaction-online) to the newly created validator wallet.

Once the wallet is deployed, place 2 files on the validator node:

- `/ton-node/configs/${VALIDATOR_NAME}.addr` should contain validator multisignature wallet address in form `X:XXX...XXX` (the folder on the host is `rustnet.ton.dev/docker-compose/ton-node/configs`) 
- `/ton-node/configs/keys/msig.keys.json` should contain validator multisignature custodian's keypair (the folder on the host is `rustnet.ton.dev/docker-compose/ton-node/configs/keys/`)

The node will use the wallet address and the keys provided to it to generate election requests each validation cycle.

> **Note**: If the validator wallet requires more than 1 custodian signature to make transactions, make sure each transaction sent by the validator node is [confirmed](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/safemultisig#47-create-transaction-confirmation-online) by the required amount of custodians.
    
## 6. Configure DePool

For a DePool validator it is necessary to deploy a [DePool](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool) contract to workchain `0`.

You can use [TONOS-CLI](https://github.com/tonlabs/tonos-cli) for this purpose. It should be [configured](https://github.com/tonlabs/tonos-cli) to connect to the main.ton.dev network.

Refer to [this document](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#depool) for the detailed DePool creation procedure, or follow the links in the short guide below:

1. [Obtain contract code](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#2-prepare-depool-and-supporting-smart-contracts) from the repository.
2. Generate [deployment keys](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#3-generate-deployment-keys).
3. Calculate [contract addresses](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#4-calculate-contract-addresses).
4. [Send tokens](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#5-send-coins-to-the-calculated-addresses) to the calculated addresses.
5. [Deploy contracts](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#6-deploy-contracts). Make sure to specify your validator wallet in the DePool contract at this step.
6. Configure DePool [state update method](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#7-configure-depool-state-update-method).

Once DePool is successfully deployed and configured to be regularly called to update its state, you can [make stakes](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#8-make-stakes) in it. Note that validator stakes must always exceed [validator assurance](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#61-deploy-depool-contract-to-the-basechain), otherwise DePool will not participate in elections.

Also note, that DePool and supporting contracts balance should be [monitored and kept positive at all times](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#11-maintain-positive-balance-on-depool-and-supplementary-contracts).

Once the validator wallet and the DePool are deployed, place 3 files on the validator node:

- `/ton-node/configs/${VALIDATOR_NAME}.addr` should contain validator multisignature wallet address in form `0:XXX...XXX` (the folder on the host is `rustnet.ton.dev/docker-compose/ton-node/configs`)
- `/ton-node/configs/keys/msig.keys.json` should contain validator multisignature custodian's keypair (the folder on the host is `rustnet.ton.dev/docker-compose/ton-node/configs/keys/`)
- `/ton-node/configs/depool.addr` should contain DePool address in form `0:XXX...XXX`  (the folder on the host is `rustnet.ton.dev/docker-compose/ton-node/configs`)


The script generating validator election requests (directly through multisig wallet, or through DePool, depending on the setting selected on step 2.1) will run regularly, once the necessary addresses and keys are provided.


# Stopping, restarting and deleting the RUST Node

> **Note**: call docker-compose commands from the `rustnet.ton.dev/docker-compose/ton-node`  folder.

To stop the node use the following command:

    docker-compose stop

To restart a stopped node use the following command:

    docker-compose restart


# Logging
## During deployment

It is highly recommended to record the full log during node deployment:

    ./deploy.sh 2>&1 | tee ./deploy.log

The log is saved to the `rustnet.ton.dev/scripts/` folder next to the deployment script and can be useful for troubleshooting.

## During operation

When operational, the node keeps a number of logs in the `rustnet.ton.dev/docker-compose/ton-node/logs` folder.

Logs are generated with **log4rs** framework. For detailed documentation on it refer to https://docs.rs/log4rs/1.0.0/log4rs/.

Logging configuration is determined by the `rustnet.ton.dev/docker-compose/ton-node/configs/log_cfg.yml` file. By default is contains the recommended configuration for the Rust node.
    
```
refresh_rate: 30 seconds

appenders:
  stdout:
    kind: console
    encoder:
      pattern: "{d(%s.%f)} {l} [{h({t})}] {I}: {m}{n}"

  stdout_ref:
    kind: console
    encoder:
      pattern: "{f}:{L} {l} [{h({t})}] {I}: {m}{n}"

  logfile:
    kind: file
    path: "/ton-node/logs/output.log"
    encoder:
      pattern: "{d(%s.%f)} {l} [{h({t})}] {I}: {m}{n}"

  rolling_logfile:
    kind: rolling_file
    encoder:
      pattern: "{d(%Y-%m-%d %H:%M:%S.%f)} {l} [{h({t})}] {I}: {m}{n}"
    path: /ton-node/logs/output.log
    policy:
      kind: compound
      trigger:
        kind: size
        limit: 50 gb
      roller:
        kind: fixed_window
        pattern: '/ton-node/logs/output_{}.log'
        base: 1
        count: 1

  tvm_logfile:
    kind: file
    path: "target/log/tvm.log"
    encoder:
      pattern: "{m}{n}"

root:
  level: info
  appenders:
    - rolling_logfile

loggers:
  # node messages
  ton_node:
    level: trace
  boot:
    level: trace
  sync:
    level: trace

  # adnl messages
  adnl:
    level: info

  overlay:
    level: info

  rldp:
    level: info

  dht:
    level: info

  # block messages
  ton_block:
    level: debug

  # block messages
  executor:
    level: debug

  # tvm messages
  tvm:
    level: info

  librdkafka:
    level: info

  validator:
    level: debug

  catchain:
    level: debug

  validator_session:
    level: debug
```

The currently configured targets are the following:

`ton_node`: node-related messages, except initial boot and sync, block exchange with other nodes

`boot`: initial boot messages, creation of trusted key block chain, loading blockchain state

`sync`: node synchronization  - loading a certain number of most recent blocks

`adnl`: messages of the ADNL protocol

`overlay`: messages of the overlay protocol

`rldp`: messages of the RLDP protocol

`dht`: messages of the DHT protocol

`ton_block`: messages of the block structures library, logs are turned on in debug

`executor`: messages of the smart contract execution library, logs are turned on in debug 

`tvm`: ton virtual machine messages, logs are turned on in debug

`librdkafka`: kafka client library messages

`validator`: top level consensus protocol messages

`catchain`: low level consensus protocol messages

`validator_session`: mid level consensus protocol messages


# Troubleshooting

Here are some solutions to frequently encountered problems.

## 1. Couldn’t connect to Docker daemon at http+docker://localhost

This error occurs in two cases. Either the docker daemon isn't running, or current user doesn't have rights to access docker.

You can fix the rights issue either by running relevant commands as the superuser or adding the user to the `docker` group: 

    sudo usermod -a -G docker $USER

Make sure to restart the system or log out and back in, for the new group settings to take effect.


## 2. thread 'main' panicked error when checking node synchronization

The following error may occur for a short time immediately after node deployment when attempting to [check synchronization](#4-check-node-synchronization):

    thread 'main' panicked at 'Can't create client: Os { code: 111, kind: ConnectionRefused, message: "Connection refused" }', bin/console.rs:454:59

Currently this is expected behavior, unless it persists **for more than a few minutes**. If it does persist, check network status at https://rustnet.ton.live/, and, if the network is up and running, review [deployment logs](#during-deployment) for errors.


## 3. Error executing command when checking node synchronization

The following error may occur for up to an hour after node deployment when attempting to [check synchronization](#4-check-node-synchronization):

    Error executing command: Error receiving answer: early eof bin/console.rs:296

Currently this is expected behavior, unless it persists **for more than one hour**. If it does persist, check network status at https://rustnet.ton.live/, and, if the network is up and running, review [deployment logs](#during-deployment) for errors.


## 4. Cannot stop/restart/remove node container

Make sure you are running all docker-compose commands from the `rustnet.ton.dev/docker-compose/ton-node` folder.


## 5. DePool state not updating

It's recommended to send at least two [ticktocks](https://github.com/tonlabs/ton-labs-contracts/tree/master/solidity/depool#7-configure-depool-state-update-method) while the elections are open.
For rust node you can use the [provided](https://github.com/tonlabs/rustnet.ton.dev/blob/main/docker-compose/ton-node/scripts/send_depool_tick_tock.sh) ticktock script, which sends 5 ticktocks after the elections open.
