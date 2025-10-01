+++
title = "Create your own l2 rollup testnet with op-stack and celestia DA"

[taxonomies]
tags = ["tutorial"]
+++

## 1. Introduction


Within the rapidly expanding Ethereum ecosystem, Layer 2 rollups have emerged as the most effective approach to scaling—reducing transaction costs, increasing throughput, and enabling new possibilities for decentralized applications.

However, for many developers, the process of deploying a rollup has long seemed daunting:
- Excessive boilerplate code,
- Numerous interdependent components, and
- Limited transparency into the underlying mechanics.

The OP Stack addresses these challenges directly.

As a modular, production-grade framework for building Ethereum-equivalent rollups, the OP Stack not only powers the Optimism mainnet but is also openly available for anyone to adapt and deploy.

This guide is designed for developers who wish to:
- Gain a deeper understanding of how rollups such as Optimism operate,
- Experiment by deploying their own Layer 2 chain, or
- Explore Ethereum’s infrastructure at a more technical level.

## 2. Quick Overview: OP Stack Components
Before we get hands-on, let’s take a moment to understand the core building blocks of an OP Stack rollup.

Just like Ethereum has a consensus client (e.g. Lighthouse, Prysm) and an execution client (e.g. Geth, Nethermind), the OP Stack has similar components — but designed for running a Layer 2 chain.

Here’s a quick overview:

#### Execution Client → op-geth
The execution client is responsible for:

- Executing smart contracts
- Maintaining the state of the chain
- Handling the Ethereum JSON-RPC interface
- In the OP Stack, the execution client is a fork of Geth called op-geth.

Think of this as the “EVM Brain” of your L2 chain.

#### Consensus Client → op-node
The consensus client is responsible for:

- Tracking L1 block (on Ethereum Sepolia in our case)
- Coordinating with the execution client using the Engine API
- Determining the canonical L2 chain based on inputs from L1
- This is handled by op-node — the core driver of your rollup.

It watches Ethereum L1, processes L2 inputs, and drives op-geth to produce L2 blocks.

#### Batcher → op-batcher
Once your rollup is producing blocks, you need to post those blocks to Ethereum L1.

The batcher:

- Collects L2 transaction data from op-geth
- Publishes it to L1 in a special contract
- Ensures data availability and verifiability
- This is what makes rollups secure: the data lives on Ethereum.

#### Proposer → op-proposer
The proposer is responsible for:

- Submitting state roots of your L1 chain to L1
- Updating the Output Oracle smart contract
- Enabling withdrawals from L2 back to L1
- Without the proposer, your chain can’t finalize.



## Setting Up the Environment
Now that you understand the major components, it’s time to roll up our sleeves and set up everything you’ll need to spin up your own L2 chain.

We’ll begin by cloning the relevant repositories, installing dependencies, and preparing the core binaries.

Cloning the Optimism Monorepo
First, clone the main Optimism Monorepo, which contains the OP Stack source code.

```bash
git clone https://github.com/ethereum-optimism/optimism.git
```

Move into the cloned directory:

```bash
cd optimism
```

### Build Core Binaries
After installing dependencies, you’ll need to build some important binaries:

```bash
make op-node op-batcher op-proposer
```
This will compile and prepare:

`op-node` (Consensus Client)
`op-batcher` (Batcher Service)
`op-proposer` (Proposer Service)

All these binaries will be available in the ./bin folder inside your Optimism directory.

Clone the Execution Client (op-geth)
The Execution Client (modified Geth) lives in a separate repository.

Clone it:

```bash
git clone https://github.com/ethereum-optimism/op-geth.git
```

Move into the op-geth directory:

```bash
cd op-geth
```

And build the geth binary:
```bash
make geth
```

This will produce a build/bin/geth binary, which you’ll use to run your L2 Execution Client.


### Install Foundry Dependencies
If you haven’t already installed Foundry, install it first:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```


## 4. Setting Up Wallets and Environment Variables
Before we can launch any part of the chain, we need special wallets for different roles — Admin, Batcher, Proposer, and Sequencer.

We’ll also prepare a .envrc file to easily load these credentials into our shell.

Let’s go step-by-step.

### Generate Wallets
Inside your cloned optimism directory, run the wallet generation script:

```bash
./packages/contracts-bedrock/scripts/getting-started/wallets.sh
```
This script will generate:

Admin account (for deployments)
Batcher account (for publishing transaction batches)
Proposer account (for posting state roots)
Sequencer account (for block production)
Copy Wallet Details
The output will look something like this:

```bash
Copy the following into your .envrc file:

# Admin address
export GS_ADMIN_ADDRESS=0x...
export GS_ADMIN_PRIVATE_KEY=0x...
# Batcher address
export GS_BATCHER_ADDRESS=0x...
export GS_BATCHER_PRIVATE_KEY=0x...
# Proposer address
export GS_PROPOSER_ADDRESS=0x...
export GS_PROPOSER_PRIVATE_KEY=0x...
# Sequencer address
export GS_SEQUENCER_ADDRESS=0x...
export GS_SEQUENCER_PRIVATE_KEY=0x...
```

### Fund Your Wallets
Since we’re deploying on Sepolia testnet, you’ll need some Sepolia ETH to fund your wallets.

Recommended funding amounts:

Admin — 0.5 Sepolia ETH
Batcher — 0.1 Sepolia ETH
Proposer — 0.2 Sepolia ETH
Sequencer — No ETH needed (it doesn’t send transactions)
You can get Sepolia ETH from public faucets or ask a friend/testnet provider.

### Installing and Set Up direnv
To load these wallet variables automatically, we’ll use a tool called direnv.

Install it:

```bash
brew install direnv || apt-get install direnv
```

Create `.envrc` file
Inside your optimism folder, create a file called `.envrc`.

Paste the exported environment variables into it, like this:


```bash
# Admin account
export GS_ADMIN_ADDRESS=0xYourAdminAddress
export GS_ADMIN_PRIVATE_KEY=0xYourAdminPrivateKey
# Batcher account
export GS_BATCHER_ADDRESS=0xYourBatcherAddress
export GS_BATCHER_PRIVATE_KEY=0xYourBatcherPrivateKey
# Proposer account
export GS_PROPOSER_ADDRESS=0xYourProposerAddress
export GS_PROPOSER_PRIVATE_KEY=0xYourProposerPrivateKey
# Sequencer account
export GS_SEQUENCER_ADDRESS=0xYourSequencerAddress
export GS_SEQUENCER_PRIVATE_KEY=0xYourSequencerPrivateKey
# L1 Sepolia RPC URL
export L1_RPC_URL="<https://eth-sepolia.g.alchemy.com/v2/your-api-key>"
# Chain IDs and Block Times
export L1_CHAIN_ID="11155111"
export L2_CHAIN_ID="420"
export L1_BLOCK_TIME=12
export L2_BLOCK_TIME=2
```


Make sure to replace `your-api-key` with your real alchemy, infura or quiknode endpoint for sepolia.

Load the Environment
Once `.envrc` is ready, tell `direnv` to load it:

```bash
direnv allow
```

If everything worked, you should see output like:
```bash
direnv: loading .envrc
direnv: export +GS_ADMIN_ADDRESS +GS_ADMIN_PRIVATE_KEY +... (and so on)
```
If you don’t see this, make sure your .envrc file is properly formatted and try direnv allow again.

## 5. Configuring the Rollup
Now that you have your environment and wallets ready, it’s time to set up the actual configuration for your L2 rollup.

This configuration defines important parameters like block times, chain IDs, addresses, and other system settings.

We’ll generate a basic config automatically, which you can later customize as needed.

Navigate to the `contracts-bedrock` Package
From your optimism monorepo root, move into the contracts-bedrock package:

```bash
cd packages/contracts-bedrock
```

Inside the contracts-bedrock package, install the dependencies:

```bash
forge install
```

This will install all the necessary libraries needed for smart contract deployment and testing.

Generate the getting-started.json Config
Run the provided configuration script:
```bash
./scripts/getting-started/config.sh
```

This will create a new file:
```bash
deploy-config/getting-started.json
```
This file will contain all the basic chain configuration based on your environment variables.

Example: What getting-started.json Looks Like
Here’s a simplified example:

```json
{
  "l1ChainID": 11155111,
  "l2ChainID": 420,
  "l2BlockTime": 2,
  "l1BlockTime": 12,
  "maxSequencerDrift": 600,
  "sequencerWindowSize": 3600,
  "channelTimeout": 300,
  "p2pSequencerAddress": "0xYourSequencerAddress",
  "batchSenderAddress": "0xYourBatcherAddress",
  "l2OutputOracleProposer": "0xYourProposerAddress",
  "l2OutputOracleChallenger": "0xYourAdminAddress",
  ...
}
```

## 6. Deploying L1 Contracts Using `op-deployer`
This is a crucial step — these contracts form the backbone of your rollup system, including bridges, oracles, and system configuration.

We’ll use the op-deployer tool to handle this process.

`op-deployer` is a command-line tool built by the Optimism team to make deploying OP Stack contracts simple and repeatable.

It handles:

- Contract deployments
- Genesis and rollup config generation
- Chain initialization steps
- Using op-deployer also ensures your deployment follows OP Stack standards, making it future-proof if you want to join the Superchain later.

#### Install `op-deployer`

Make sure you use a release version.

Inside your optimism directory, the op-deployer binary should already be built when you ran make.

If not, you can manually build it:

```bash
cd optimism/op-deployer
just build
```
Ensure that `bin/op-deployer` exists.

#### Initialize a `.deployer` Directory
Now, we’ll create a working directory where op-deployer will manage deployment state.

From the optimism root directory, run:

```bash
./bin/op-deployer init --l1-chain-id 11155111 --l2-chain-ids 420 --workdir .deployer
```

`11155111` → Sepolia L1 chain ID

`420` → L2 chain ID for our rollup

`.deployer` → Directory where deployment config and state files will be stored

This command creates:

`.deployer/intent.toml` (your deployment config)

`.deployer/state.json` (populated after deploy)

#### Customize Your Intent File
You can open `.deployer/intent.json` to tweak settings like:

- Owner addresses
- Fee recipient addresses
- Governance settings
For now, default settings are fine for a testnet rollup.

#### Deploy Contracts to L1
Now deploy the contracts to Sepolia:

```bash
./bin/op-deployer apply --workdir .deployer --l1-rpc-url $L1_RPC_URL --private-key $GS_ADMIN_PRIVATE_KEY
```

`$L1_RPC_URL` → your Sepolia RPC URL (Alchemy, Infura, etc.)

`$GS_ADMIN_PRIVATE_KEY` → Admin wallet’s private key you generated earlier


- Contracts like the L1StandardBridge, SystemConfig, OutputOracle etc. are deployed on Sepolia
- Deployer records all deployed contract addresses into `.deployer/state.json`
- The deployment flow follows the config you created earlier
- If everything works, you’ll see success messages with deployed addresses.



If something fails (like “out of gas” errors), check:
Fund balances on Admin account
Correct environment variables loaded

#### Result — Deployed State
After a successful deployment, you will have:

`.deployer/state.json` — contract addresses, deployment data

`.deployer/` directory — full deployment snapshot

We will use this data in the next steps to generate genesis and initialize the L2 nodes.

## 7. Generating the Genesis and Rollup Config Files
Now that your smart contracts are deployed on Sepolia, it’s time to generate the core configuration files your chain will need to actually run:

`genesis.json` → for the Execution Client (op-geth)

`rollup.json` → for the Consensus Client (op-node)


#### Generate genesis.json
The `genesis.json` file defines the initial state for your Execution Client.

It tells op-geth:

What the first block looks like
Who the system accounts are
What settings to apply at startup
To generate it, run:

```bash
./bin/op-deployer inspect genesis --workdir .deployer <L2_CHAIN_ID> > .deployer/genesis.json
```
In our case:
```bash
./bin/op-deployer inspect genesis --workdir .deployer 420 > .deployer/genesis.json
```
This command reads from .deployer/state.json and produces .deployer/genesis.json.

#### Generate rollup.json
The rollup.json file defines the rollup configuration for your Consensus Client.

It tells `op-node`:

Which contracts to watch on L1
How to verify blocks
Timing parameters and security assumptions
To generate it, run:

```bash
./bin/op-deployer inspect rollup --workdir .deployer <L2_CHAIN_ID> > .deployer/rollup.json
```
In our case:
```bash
./bin/op-deployer inspect rollup --workdir .deployer 420 > .deployer/rollup.json
```

You now should have:

`.deployer/genesis.json`
`.deployer/rollup.json`

We’ll copy these files to the right places when initializing op-geth and op-node in the next steps.

## 8. Running the Core Clients
With your contracts deployed and configuration files generated, it’s time to start the two main engines of your rollup:

Execution Client (op-geth)
Consensus Client (op-node)
These two clients will work together to drive your L2 rollup.

Let’s go step-by-step.

#### Initialize op-geth
First, we need to initialize the Execution Client (op-geth) using the genesis.json we created.

Navigate to your op-geth directory:

```bash
cd ~/op-geth
mkdir datadir
```

Initialize op-geth:

```bash
build/bin/geth init --state.scheme=hash --datadir=datadir ../optimism/.deployer/genesis.json
```
This tells op-geth:

Use the starting block specified in genesis.json
Store blockchain data in the datadir/ folder
#### Start op-geth
Now, start the Execution Client:

```bash
./build/bin/geth \
  --datadir ./datadir \
  --http \
  --http.corsdomain="*" \
  --http.vhosts="*" \
  --http.addr=0.0.0.0 \
  --http.api=web3,debug,eth,txpool,net,engine \
  --ws \
  --ws.addr=0.0.0.0 \
  --ws.port=8546 \
  --ws.origins="*" \
  --ws.api=debug,eth,txpool,net,engine \
  --syncmode=full \
  --gcmode=archive \
  --nodiscover \
  --maxpeers=0 \
  --networkid=42069 \
  --authrpc.vhosts="*" \
  --authrpc.addr=0.0.0.0 \
  --authrpc.port=8551 \
  --authrpc.jwtsecret=./jwt.txt \
  --rollup.disabletxpoolgossip=true
```

Important settings:

`-gcmode=archive` → important for Sequencers because op-proposer needs full state.
`-authrpc.jwtsecret=./jwt.txt` → this JWT secret is used to authenticate between op-geth and op-node.
If this runs successfully, your Execution Client is live!

Step 3: Start op-node

```bash
cd ~/optimism/op-node
./bin/op-node \
  --l2=http://localhost:8551 \
  --l2.jwt-secret=./jwt.txt \
  --sequencer.enabled \
  --sequencer.l1-confs=5 \
  --verifier.l1-confs=4 \
  --rollup.config=../.deployer/rollup.json \
  --rpc.addr=0.0.0.0 \
  --rpc.port=9545 \
  --p2p.disable \
  --rpc.enable-admin \
  --p2p.sequencer.key=$GS_SEQUENCER_PRIVATE_KEY \
  --l1=$L1_RPC_URL \
  --l1.rpckind=$L1_RPC_KIND
```


This will:

- Connect op-node to your op-geth
- Start producing L2 blocks
- Monitor L1 (Sepolia) for updates
- At this point, you should start seeing L2 block production happening live in the logs!

How to Know It’s Working
If everything is correctly set up:

op-geth will show new blocks being imported
op-node will show new L2 blocks being proposed and finalized
You now have a working, local Sequencer node!

## 9. Running Supporting Services (op-batcher and op-proposer)
Now that your Execution Client (op-geth) and Consensus Client (op-node) are live and producing L2 blocks,

You still need two critical background services to complete the full rollup lifecycle:

`Batcher (op-batcher)` → Posts transaction data to L1
`Proposer (op-proposer)` → Posts state roots to L1 (for withdrawals and finality)


##### 1. Start the Batcher (op-batcher)
The batcher collects blocks and transaction data from your L2 chain, bundles them into batches, and sends them to the Batch Inbox contract on Sepolia.

Without the batcher, your rollup won’t post any data to Ethereum.

```bash
cd ~/optimism/op-batcher
./bin/op-batcher \
  --l2-eth-rpc=http://localhost:8545 \
  --rollup-rpc=http://localhost:9545 \
  --poll-interval=1s \
  --sub-safety-margin=6 \
  --num-confirmations=1 \
  --safe-abort-nonce-too-low-count=3 \
  --resubmission-timeout=30s \
  --rpc.addr=0.0.0.0 \
  --rpc.port=8548 \
  --rpc.enable-admin \
  --max-channel-duration=25 \
  --l1-eth-rpc=$L1_RPC_URL \
  --private-key=$GS_BATCHER_PRIVATE_KEY
```

Now, your batcher is running!

It will:

Monitor your op-geth
Submit batched L2 transactions to Sepolia (L1)

##### 2. Start the Proposer (op-proposer)
The proposer submits L2 state roots to the L2OutputOracle contract on Sepolia.

This enables users to withdraw funds back to L1 eventually.

Open another terminal window.

Navigate to the proposer directory:

```bash
cd ~/optimism/op-proposer
./bin/op-proposer \
  --poll-interval=12s \
  --rpc.port=8560 \
  --rollup-rpc=http://localhost:9545 \
  --l2oo-address=$(cat ../packages/contracts-bedrock/deployments/getting-started/.deploy | jq -r .L2OutputOracleProxy) \
  --private-key=$GS_PROPOSER_PRIVATE_KEY \
  --l1-eth-rpc=$L1_RPC_URL
```

Now, your proposer is live!

It will:

Watch your L2 chain
Post finalized L2 outputs to L1


## 10. Connecting Your Wallet to Your Rollup
Now that your rollup is fully running — producing blocks, posting batches, and finalizing states —

It’s time to connect your wallet (like MetaMask) and interact with your L2 network directly.

This is where it starts feeling real — sending transactions, deploying contracts, and seeing activity on your very own chain!

##### Add Your Rollup as a Custom Network in MetaMask
Open MetaMask and add a new custom network manually.

Here’s the information you’ll need:

Network Name → OP Stack Rollup (or any name you like)

New RPC URL → http:localhost:8545

Chain ID → 420 (or whatever you configured in .envrc)

Currency Symbol → ETH

Block Explorer URL → (leave blank for now)

Save the network.

Your MetaMask should now point to your local Rollup at localhost:8545.

##### Confirm You Are Connected
Once added:

You should see your wallet connected to your L2 chain.
Your wallet address should appear exactly the same as it does on Sepolia (since private keys are the same).
But your balance will probably be 0 ETH on L2 (because bridging hasn’t happened yet).
Don’t worry!

We’ll fix that in the next section.

## 11. Bridging ETH to Your Rollup
Now that your wallet is connected to your L2 rollup, you’ll notice you have 0 ETH on your rollup network.

To send transactions or deploy contracts, you need ETH on your L2 chain.

How do you get ETH onto your rollup?

By bridging it through the L1 Standard Bridge contract you deployed earlier.

Let’s do it step-by-step.

##### Step 1: Find the L1 Bridge Contract Address
Navigate to your contracts-bedrock package:

```bash
cd ~/optimism/packages/contracts-bedrock
```
Get the address of the L1StandardBridgeProxy that was deployed when you ran op-deployer:
```bash
cat deployments/getting-started/.deploy | jq -r .L1StandardBridgeProxy
```
This will output the address you need to send ETH to.

Save this address.

##### Send Sepolia ETH to the Bridge Contract
Open your MetaMask (or any wallet connected to Sepolia) and send a small amount of Sepolia ETH (like 0.05 or 0.1 ETH) to the L1StandardBridgeProxy address you just got.

This will trigger a deposit into your rollup.

Important tips:

Don’t send large amounts during testing.
Wait for the Sepolia transaction to be confirmed.

##### Wait for Bridging to Complete
After sending ETH to the bridge:

- Your L1 deposit must be finalized by the rollup.
- The op-node will observe the L1 deposit.
- After a few blocks (usually 2–5 minutes), you’ll see the ETH minted on L2.
- You can check logs in your op-node and op-geth terminals — you should see "deposit finalized" or similar logs.

Once completed, your wallet on the Rollup network will show the bridged ETH.

## 12. Testing Your Rollup
Now that you have bridged ETH onto your rollup, you’re finally ready to use your new blockchain just like any other EVM chain!

You can now send transactions, deploy smart contracts, and observe your rollup in action.

Let’s walk through the basic tests you should do.

#### Send a Test Transaction
Use your MetaMask (or any wallet connected to your L2 network) and:

Send a small amount of ETH to another address (or even back to yourself).
Confirm the transaction in MetaMask.
Within a few seconds, you should see:

The transaction is confirmed in your wallet.
Logs appearing inside your op-geth and op-node terminals showing the transaction being executed.
If this works, it means:

Your RPC endpoint (localhost:8545) is live
Your execution and consensus clients are processing new transactions
Your rollup is healthy


#### Deploy a Smart Contract
For a full test, let’s also deploy a simple smart contract!

You can use tools like Remix IDE, Hardhat, or Foundry.

Here’s a basic way using Remix:

Open Remix.
Set the environment to “Injected Web3” → Connects to MetaMask.
Make sure MetaMask is connected to your OP Stack Rollup (localhost:8545).
Write a very simple contract like:

```bash
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract HelloWorld {
    string public message = "Hello, Rollup!";
}
```

Compile and Deploy it.
✅ You should see:

Deployment transaction confirmed on your L2
Contract address available
Ability to call functions (message) instantly
What Are You Testing Here
You are verifying:

Your rollup can accept and process user-initiated transactions
Your chain supports smart contract deployment
Gas metering, state updates, and storage are working properly
This is full EVM functionality — on your own L2 rollup!
