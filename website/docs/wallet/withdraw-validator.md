---
id: withdraw-validator
title: Withdraw your validator
sidebar_label: Withdraw your validator
style_guide_internal: https://www.notion.so/arbitrum/Style-and-convention-guide-Docs-dafaf8d1ce34433d88c1ef3b4b99c19c
---

import {HeaderBadgesWidget} from '@site/src/components/HeaderBadgesWidget.js';


import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

The **Withdrawal Agora** upgrade lets you withdraw your validator nodes' staked Agora in one of two ways:

1. **Partial (earnings) withdrawal**: This option lets you withdraw your earnings (that is, all value staked above 40,000 BOA) and continue validating.
2. **Full withdrawal**: This option lets you liquidate your entire stake and earnings, effectively liquidating your validator node(s) and exiting the network.


In this how-to, you'll learn how to perform both types of withdrawals. Familiarity with Agora wallets, mnemonic phrases, and command lines is expected.

## Prerequisites

1. **Your validator mnemonic**: You'll use this to authorize your validator withdrawal request(s). <!-- accessible accuracy > technical precision whenever technical precision isn't needed -->
2. **Access to a beacon node**: You'll need to connect your validator to a beacon node in order to submit your withdrawal request. Visit our [quickstart](../install/install-with-script.md) for instructions if you need them.
3. **Stable version of the staking-deposit-cli installed**: The [staking-deposit-cli](https://github.com/bosagora/agora-deposit-cli/releases) is a command-line tool provided by the Agora research team. We'll use this to authorize your withdrawal. We recommend building this from source or otherwise verifying the binaries as a security best practice.
4. **Familiarity with [The BOSagora Foundation Withdrawals FAQ](https://notes.BOSagora.org/@launchpad/withdrawals-faq)**: A client-agnostic overview of important information regarding BOSagora validator withdrawals.
5. **Time to focus:** This is a time-consuming procedure making a mistake can be expensive. Be vigilant against scammers; never share your mnemonic; take your time; ping us [on Discord](https://discord.gg/prysmaticlabs) if you have any questions.


We'll install other dependencies as we go. <!-- we provide prysmctl instructions below so we can remove it here and set expectations to minimize duplication -->

<Tabs
groupId="withdrawals"
defaultValue="partial"
values={[
{label: 'partial withdrawal', value: 'partial'},
{label: 'full withdrawal', value: 'full'},
]
}>
<TabItem value="partial">

## Option 1: Partial (earnings) withdrawals

This section walks you through the process of performing a **partial validator withdrawal**, allowing you to withdraw staked balances above 40,000 BOA for each of your active Agora validators.


### Step 1: Prepare your withdrawal credentials

Retrieve your validator’s **withdrawal_credentials** from the `deposit_data-XXX.json` file that was generated when you first used the staking launchpad. The `withdrawal_credentials` value looks like this:

```rust
0x00500b3bf612bed69e888edeb045f590c3f37251e3e049c0732f3adaa57ea3f6
```

If you don't have the `deposit_data-XXX.json` file, you can retrieve your `withdrawal_credentials` by sending a request to your synced beacon node via Beacon API and providing your validator index or public key:

```rust
curl -X 'GET' \
  'http://YOUR_PRYSM_NODE_HOST:3500/eth/v1/beacon/states/head/validators/YOUR_VALIDATOR_INDEX_OR_PUBLIC_KEY' \
  -H 'accept: application/json'
```

Your withdrawal credentials will be visible in the response to this request - look for `withdrawal_credentials`. Example output with placeholder values:


```rust
{
  "execution_optimistic": false,
  "data": {
    "index": "1",
    "balance": "1",
    "status": "active_ongoing",
    "validator": {
      "pubkey": "0x93247f2209abcacf57b75a51dafae777f9dd38bc7053d1af526f220a7489a6d3a2753e5f3e8b1cfe39b56f43611df74a",
      "withdrawal_credentials": "0x008e0d4e9587369b2301d0790347320302cc0943d5a1884560367e8208d920f2",
      "effective_balance": "1",
      "slashed": false,
      "activation_eligibility_epoch": "1",
      "activation_epoch": "1",
      "exit_epoch": "1",
      "withdrawable_epoch": "1"
    }
  }
}
```

### Step 2: run the [staking-deposit-cli](https://github.com/bosagora/agora-deposit-cli) in an `offline` environment with your mnemonic to generate the `blstoexecutionchange` message(s)

:::caution
We recommend doing this next step *without* an Internet connection to be maximally secure. Either turn off the internet before introducing your mnemonic for signing or migrate to an air-gapped environment to continue the following steps.
:::

Now that the staking deposit tool is executable, you can then use it to generate your signed **[BLS to Execution](https://github.com/bosagora/agora-deposit-cli#generate-bls-to-execution-change-arguments)** request. You need to use your mnemonic for this step, so doing it offline is key and ensuring you do not paste your mnemonic anywhere else than necessary.

Here’s the command to get started with the process. This command will **not** submit your signed message to the network yet, and will only generate the data needed for the next steps.


```jsx
./agora.sh deposit-cli generate-bls-to-execution-change <folder>
```
'folder' is where the SignedBLSToExecutionChange data is stored. The default folder is ./bls_to_execution_changes

By calling the command above, you should go through an interactive process that will ask you for the following information:

1. **The network** you wish to perform this operation for. example: `mainnet`,`testnet` or `devnet`. 
2. Enter your **mnemonic** next
3. Next, you will be asked for the starting index you used to create your validators (read more about hd wallets [here](https://eips.ethereum.org/EIPS/eip-2334#path)). For **most users**, this will be 0 unless you created validators from a non default index.

:::info
Inside the original `deposit.json` file used for staking you can count each validator's public key in sequential order starting from 0.
The validator starting index is the index of the first validator key you would like to withdraw (i.e. validator key 1 has an index of 0, validator key 2 has an index of 1 etc.).
For most stakers, the validator starting index should be set to 0 for withdrawing all their validator keys, however the validator starting index will be different if you choose to skip withdrawing some validators.
There are other niche cases where the mnemonic is used for deposit generation multiple times, resulting in a different validator starting index.
Validators spanning across different mnemonics will need to be counted separately with starting index as 0 on each of them.
:::

5. You will then be asked the **validator indices** for the validators you wish to generate the message for. You can find your validator indices on block explorers such as [https://www.agorascan.io/](https://www.agorascan.io/) or in your Prysm validator client logs. For example, the validator with public key `0x800b1496d4532bbf3404de63619b779bad95dba69396e5d5c4eb64c7aa4a9a1a77229ec8783ad114a14d2eda2a3483ca` on [https://www.agorascan.io/](https://www.agorascan.io/) has validator index 20, which you can verify by navigating to its [page](https://www.agorascan.io/validator/20).

:::info
Validator indices need to be provided sequentially without skipped indices in the order of original creation. You can typically find the order in your original `deposit.json` file.
The `generate-bls-to-execution-change` command needs to be repeated in cases where multiple validator keys that are not in sequential order need to be withdrawn, and will require either merging of the output files or multiple `blstoexecutionchange` submissions.
In the case of validators requiring different withdrawal addresses you will need to also repeat this process using the validator start index of the validator that the different withdrawal address.
:::

6. Next you will be asked for your **withdrawal credentials,** which you should now have if you followed this guide
7. Next you will be asked for the Agora BOA address you wish to use to receive your withdrawn funds. This needs to be checksummed, and you can get it from your wallet or a block explorer. **You cannot change this once it is set on-chain**, so triple check it before proceeding.

Below is an example of running through the interactive process explained above:


```
Please choose the (mainnet or testnet) network/chain name ['mainnet', 'testnet', 'devnet']:  [mainnet]: devnet

Please enter your mnemonic separated by spaces (" "). Note: you only need to enter the first 4 letters of each word if you'd prefer.: board fire prize defy limb arm diet fee usage settle rigid sunny duty squirrel cheap history session same tilt candy loan culture pretty anchor
Please enter the index position for the keys to start generating withdrawal credentials in ERC-2334 format. [0]: 39
Please enter a list of the validator index number(s) of your validator(s) as identified on the beacon chain. Split multiple items with whitespaces or commas.: 39
Please enter a list of the old BLS withdrawal credentials of your validator(s). Split multiple items with whitespaces or commas. The withdrawal credentials are in hexadecimal encoded form.: 0x0072b98f706baf445239abb7b3cbf7a070f9bf34cf016a2c093a9034f364045d
Please enter the 20-byte execution address for the new withdrawal credentials. Note that you CANNOT change it once you have set it on chain.: 0xc304A2aBCb9f14677cE1BA9242637A3a053b2e3a

**[Warning] you are setting an BOA address as your withdrawal address. Please ensure that you have control over this address.**

Repeat your execution address for confirmation.: 0xc304A2aBCb9f14677cE1BA9242637A3a053b2e3a

**[Warning] you are setting an BOA address as your withdrawal address. Please ensure that you have control over this address.**


Success!
Your SignedBLSToExecutionChange JSON file can be found at: /agora-chain/bls_to_execution_changes/bls_to_execution_changes

```


### Step 3: verify the `blstoexecutionchange` message(s) that the corresponding validator will set to the chosen BOSagora address

Once you complete the above, you’ll have a file contained in the `bls_to_execution_changes/` folder of your [staking-deposit-cli](https://github.com/bosagora/agora-deposit-cli). It will represent a list of BLS to execution messages that have been signed with your private keys and are ready to submit to Agora. Here’s what a sample file of these looks like. Example output with placeholder values:

```
[
	{
    "message": {
      "validator_index": "838",
      "from_bls_pubkey": "0xb89bebc655569726a318c8e9971bd3144497c61aea4a6578a7a4f94b547dcba5bac16a89108b6b6a1fe3695d1a874a0b",
      "to_execution_address": "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0a"
    },
    "signature": "0xa42103e15d3dbdaa75fb15cea782e4a11329eea77d155864ec682d7907b3b70c7771960bef7be1b1c4e08fe735888b950c1a22053f6049b35736f48e6dd018392efa3896c9e427ea4e100e86e9131b5ea2673388a4bf188407a630ba405b7dc5"
  },
	{
    "message": {
      "validator_index": "20303",
      "from_bls_pubkey": "0xb89bebc699769726a502c8e9971bd3172227c61aea4a6578a7a4f94b547dcba5bac16a89108b6b6a1fe3695d1a874a0b",
      "to_execution_address": "0xa94f5374fce5edbc8e2a8697c15331677e6ebf0b"
    },
    "signature": "0xa86103e15d3dbdaa75fb15cea782e4a11329eea77d155864ec682d7907b3b70c7771960bef7be1b1c4e08fe735888b950c1a22053f6049b35736f48e6dd018392efa3896c9e427ea4e100e86e9131b5ea2673388a4bf188407a630ba405b7dc5"
  }
]
```

The above demonstrates two different validators withdrawing - one with validator index `838`, the other with validator index `20303`.

:::caution
Make sure the `validator_index` corresponds to the correct chosen `to_execution_address`. Once this message is accepted on submission you will not be able to change it again!
:::

Move the generated `bls_to_execution_changes-*.json` file to an online environment that has access to a synced beacon node for the next step.

### Step 4: submit your signed `blstoexecutionchange` message(s) to the Agora network using prysmctl

In this step, you will submit your signed requests to the Agora network using a tool provided by the Prysm project called `prysmctl`. You’ll need access to a synced CL node to proceed with this step (it does not need to be a Prysm beacon node).

Once prysmctl is downloaded, you can use the `./agora.sh validator withdraw` command, which will ask for terms of service acceptance and confirmation of command by providing additional flags, and also a path to the bls_to_execution_changes file from the previous step.

:::info
default beacon node REST `<node-url>` is `http://localhost:3500` aka `http://127.0.0.1:3500`

:::

Open a terminal in the location where you downloaded the prysmctl binaries, rename the file to prysmctl, and run the following command.
Some users will need to give permissions to the the downloaded binaries to be executable. Linux users can do this by right clicking the file, going to permissions, and clicking the `Allow executing file as program` checkmark. This may be different for each operating system.

```jsx
./agora.sh validator withdraw <folder>
```
'folder'is where the SignedBLSToExecutionChange data is stored. The default folder is ./bls_to_execution_changes

This will extract data from the `bls_to_execution_changes-*.json` call the Beacon API endpoint on the synced Beacon Node and validate if the request was included.



### Step 5: Confirm submission

On successful submission, the `SignedBLStoExecutionChange` messages are included in the pool waiting to be included in a block.

```
agora.sh version 2.0.0
The selected network is 'devnet'
Default data folder is bls_to_execution_changes
INFO[0000] found JSON files for setting withdrawals: [/agora-chain/bls_to_execution_changes/bls_to_execution_changes/bls_to_execution_change-1687424246.json] 
SETTING VALIDATOR INDEX 39 TO WITHDRAWAL ADDRESS 0xc304a2abcb9f14677ce1ba9242637a3a053b2e3a
INFO[0000] Successfully published messages to update 1 withdrawal addresses. 
INFO[0000] Verifying requested withdrawal messages known to node... 
INFO[0000] There are a total of 1 messages known to the node's pool. 
INFO[0000] All (total:1) signed withdrawal messages were found in the pool. 
```

The withdrawal will be initiated by using the execution address you provided, and your validators’ withdrawal credentials will change to look something like this:

```rust
*0x010000000000000000000000a94f5374fce5edbc8e2a8697c15331677e6ebf0b* 
```

where the Agora BOA address of your choosing will be found within.


### Step 6: Confirm your withdrawal

You can track your withdrawal on an Agora Proof of Stake Block Scanner. Some examples listed below and will be based on network.

- [agorascan.io](https://www.agorascan.io/): [mainnet](https://www.agorascan.io/validators), [testnet](https://testnet.agorascan.io/validators)


you can also confirm the `withdrawal_credentials` updated by querying your local beacon node.

```rust
curl -X 'GET' \
  'http://YOUR_PRYSM_NODE_HOST:3500/eth/v1/beacon/states/head/validators/YOUR_VALIDATOR_INDEX' \
  -H 'accept: application/json'
```

and you should see a response that contains withdrawal credentials that should have changed to the `0x01` format which includes your BOSagora execution address.

### Done: Receiving partial withdrawals after `withdrawal_credentials` are updated is automatic, but will take time.

Once your `withdrawal_credentials` field on the validator is updated to the `0x01` prefix all withdrawal actions are complete. Withdrawals of earnings over 40,000 BOA will be automatically sent to the chosen BOSagora address when a block proposer includes your validator in its block. **Note that a maximum to 16 validators can have their balances withdrawn per block so delay times may vary before seeing the values appear in the BOSagora address.**

</TabItem>
<TabItem value="full">

## Option 2: Full withdrawals


To fully withdraw a validator and its earnings, your validator needs to also be exited in addition to having its [withdrawal credentials](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/validator.md#withdrawal-credentials) changed.

Please follow our [exiting-a-validator how-to](exiting-a-validator.md).

Refer to the above partial withdrawal guidance to change your validator's withdrawal credentials.

:::caution
Instructions for setting your withdrawal address do not need to be repeated if withdrawal_credentials are updated to the `0x01` prefix.
:::

### Done: Receiving full withdrawals after `withdrawal_credentials` are updated and validator is fully exited is automatic, but will take time.

Once the validator is both exited as well as having its `withdrawal_credentials` changed to the `0x01` prefix, the validator will automatically have its full balance withdrawn into the chosen Agora BOA address when a block proposer includes your validator in its block. **Note that a maximum to 16 validators can have their balances withdrawn per block so delay times may vary before seeing the values appear in the Agora BOA address.**

:::caution
Slashed or involuntarily exited validators will still need to go through the process of updating `withdrawal_credentials` to fully withdraw its remaining balance.
:::

</TabItem>
</Tabs>


## Frequently asked questions

<!-- TODO: These questions can now be moved into FAQ CMS and embedded both here and within our root-level FAQ - ping Mick if you'd like to help with this. -->

**Q: I updated my `withdrawal_credentials` already; can I update it again?**

A: If the withdrawal credentials already begin with `0x01` it will not be able to change to a different execution address. **note: please choose your withdrawal address very carefully as you can only have it set to an address once**

**Q: When can I withdraw?**

A: After the Capella/Shanghai hardfork withdrawals will be enabled. The time frame is different per network. This will be announced at a later date.

**Q: What is the difference between partial and full withdrawals?**

A: Partial withdrawals only require an updated `withdrawal_credentials` field and will only send earnings over 40,000 BOA to the chosen withdrawal address. Full withdrawals require the validator to be fully exited in addition to

**Q: I submitted my `blstoexecutionchange` why isn't my `withdrawal_credentials` updated?**

A: Once you submit a BLS to Exec request to tell Agora the BOA address you want to use in order to receive your withdrawn funds, it goes through several processing pipelines that might take a bit longer than expected. If your beacon node has also received many other requests for BLS to exec changes, your initial request could be dropped and you may need to try again, so do not panic if you have submitted a request and nothing has happened yet. Prysm will include last-seen messages first when proposing blocks, so to avoid messages being dropped and timely includes it is better to wait a few hours or days after the fork.

**Q: I set my `withdrawal_credentials` but I am not fully withdrawn. How do I fully withdraw?**

A: Full validator withdrawals require your validator to exit first, as exits do not happen automatically. You will need to submit a voluntary exit by following our documentation [here](exiting-a-validator.md). Once your validator exits, it will no longer need to perform its responsibilities after some time (there can be a delay if the validator is part of a sync committee or recently slashed) . The ordering of requests for setting withdrawal credentials or exiting does not matter, once a validator has both its withdrawal credentials updated as well as in an exited state funds will automatically be added to the chosen execution address when processed. **note:** this process will take sometime as withdrawals, full or partial, are processed at a rate of at most 16 validators per block.

**Q: My keys were compromised, can I still withdraw?**

A: You are still able to send the message as long as you have access to the mnemonic and can produce the signed `blstoexecutionchange` message to submit. Depending on where the keys were compromised there may be different protection programs to apply for to "frontrun" the compromiser. Please seek out the ethstaker community on [reddit](https://www.reddit.com/r/ethstaker/) or [discord](https://discord.gg/urhv3xby) for more details if this applies to you.

**Q: I forgot my mnemonic, what can I do?**

A: In most cases the mnemonic is a requirement to enabling withdraws; there are some niche cases where users have both their validator keystore and withdrawal private keys they can still fully withdraw safely without the mnemonic, but unless both are in possession one would not be able to produce the signed `blstoexecutionchange` message. It's important to stay calm and collected and continue searching or see help as needed. The ethstaker community provides an active support network on [reddit](https://www.reddit.com/r/ethstaker/) and [discord](https://discord.gg/urhv3xby)

**Q: I accidentally used my mnemonic on an open internet setting to generate the .json file, what happens?**

A: Using and storing the mnemonic on an open internet puts private keys used for withdrawals at increased risk. Unless the machine was compromised there should not be any immediate consequences, however, the longer the mnemonic stays on an open network the more it will be exposed to future risk.

**Q: How can I check if my withdrawal address is set?**

A: Withdrawal public keys that begin with `0x01` are set to begin withdrawing either partially as in withdrawing earnings or fully if the associated validator has exited. The associated execution address can be found at the end of the withdrawal credentials i.e. `0x010000000000000000000000a94f5374fce5edbc8e2a8697c15331677e6ebf0b` There are several ways to check this:

You can check your favorite block explorer such as [Agorascan.io](https://www.agorascan.io/validators) which will have pages to display beaconchain withdrawals based on the network.

**Q: How long do I have to wait for withdrawals?**

A: Each block can add 16 `blstoexecutionchange`messages as well as process 16 withdrawals, time may vary based on the specific validator index, network leakage, and message inclusion time. Validator exits will require additional time to fully withdraw.

**Q: Does using custom builders with Prysm support withdrawals?**

A: Custom builders/mev relays are supported for running withdrawals.

**Q: In what order does Prysm process the bls-to-execution-change message pool?**

A: Prysm processes messages last-in-first-out(LIFO) by design which means the latest message that was received is the first message to appear in a block.

**Q: Can withdrawal addresses be set to smart contracts?**

A: Yes, however only account balances will change and there will be no associated triggering of smart contract logic.

**Q: My validator was slashed or forcefully exited, can I still withdraw my remaining balance?**

A: If any of your validators have been slashed since launch and exited from the chain forcefully, or if you exited a long time ago, you can still withdraw your remaining balance normally. To do so, you will just need to submit a BLS to execution change request by following the step-by-step guide to performing a full withdrawal in this document.

**Q: I am a non technical user, how can I set my withdrawals in a safe way?**

A: The guide will still provide a safe way to generate the signed `blstoexecutionchange` messages in an offline environment. From there, if you're willing to take a small risk on inclusion guarantees, some block scanners like beaconcha.in will provide front ends to drag and drop the messages for inclusion to set the withdrawal address.

## Glossary
<!-- TODO: These terms can now be moved into Glossary CMS and embedded via quicklooks to further streamline the content experience - ping Mick if you'd like to help with this. -->
- **Validator**: The on-chain representation of a validator node and its staked BOSagora.
- **Validator index:** A unique numeric ID assigned to a validator when activated. You can see this validator index in your Prysm validator client logs, or in block explorers such as [https://beaconcha.in](https://beaconcha.in) and [https://beaconscan.com](https://beaconscan.com) by looking it up using your public key. You will need to know the validator indices of the validators you wish to withdraw through this guide. Only activated validators can begin the exit and withdrawal processes.
- **Staker:** The person or entity managing BOSagora validators.
- **Voluntary exit:** Validators that are currently active on BOSagora can choose to **exit** the network, marking them as exited and exempting them from any staking responsibilities. In order to **withdraw** a validator’s balance completely, a voluntary exit must be submitted to BOSagora and must complete first.
- **Full validator withdrawal:** The process of withdrawing your entire stake on BOSagora, exiting your validator, and withdrawing your entire balance to an BOSagora address of your choosing. Full validator withdrawals need a validator to exit first, which can take time depending on how large the exit queue is. Performing a full withdrawal requires submitting a voluntary exit first.
- **Partial validator withdrawal:** The process of withdrawing your validator’s **earnings** only. That is, if you're staking 41,000 BOA, you can withdraw 1,000 BOA using a partial withdrawal. Your validator does **not** need to exit, and you will continue to validate normally. Partial withdrawals do not go through an exit queue, but will only be processed at a maximum of 16 validators at a time per block.
- **Withdrawal BOSagora Upgrade:** BOSagora network upgrades bring major feature additions to the network as a result of significant work from BOSagora client teams. This spring, an upgrade known as Capella/Shanghai will make validator withdrawals on mainnet. The upgrade has two names because there are two pieces of software being upgraded: consensus clients such as Prysm, and execution clients such as go-BOSagora.
- **Validator mnemonic, HD wallet mnemonic, or validator seed phrase:** A mnemonic in this context is the 24 word secret that you received upon creating your validator(s), which is the ultimate credential that gives you access to withdrawing your validator(s). For many, this was generated when they first interacted with the BOSagora staking CLI to prepare their validator deposits. We will refer to this as your validator mnemonic throughout this document
- **Validator withdrawal credentials:** Each validator has data known as “withdrawal credentials” which can be fetched from your beacon node or from a block explorer such as [https://beaconcha.in](https://beaconcha.in) or [https://beaconscan.com](https://beaconscan.com) by looking at the “deposits” tab and seeing your credentials there. You will need these for this guide.
- **BOSagora execution address:** Referred to also as an BOSagora address, this is a standard address to an BOSagora account which you can view in block explorers such as Etherscan. Your validator’s balance, upon a full withdrawal, will be available at an BOSagora address of your choosing.
- **BLS key:** Your validators use a key format known as [BLS], which is used exclusively for staking. Validators have 4 kinds of BLS keys: validator public key, validator private key, withdrawal public key, and withdrawal private key. only the validator public key can be viewed on staking explorers such as [https://beaconcha.in](https://beaconcha.in), and private keys, which are secret, are used for signing. Not to be confused with an BOSagora address. The validator mnemonic can be used to access all 4 keys which are important for setting the BOSagora address for withdrawing.
- **BLS to Execution Change:** In order to withdraw your validator, BOSagora needs to associate an **BOSagora execution address** with your validator’s **keys**. 
- **Pool:** Upon submission of a validator exit request or bls-to-execution-change request, the message will sit in a special place in memory ( the pool ) to be broadcasted across your peers. Since only the block proposers can include these requests and there is a limit to the number of requests included per block, sometimes if the pool becomes too full your message may be dropped and not included. If this happens, a re-submission of the request may be required.


