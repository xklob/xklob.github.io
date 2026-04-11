# Selected Projects

## Tarkin - Generalized Frontrunner MEV Bot

Inspired by the Paradigm [Dark Forest paper](https://www.paradigm.xyz/2020/08/ethereum-is-a-dark-forest) published in late 2020, I set out to build the so-called "cosmic horror" named in the paper - a generalized frontrunner. In the words of Dan Robinson, this bot would "look for any transaction that they could profitably frontrun by copying it and replacing addresses with their own" in order to create money out of thin air. After several frustrating iterations, numerous roadblocks, and much learning, I arrived at the first *working* version on the third attempt: [Tarkin](https://en.wikipedia.org/wiki/Grand_Moff_Tarkin).

### Capabilities

While running, Tarkin would inspect the mempool of the connected node, and upon hearing of any new transactions, would simulate those transactions *as if Tarkin had sent them*, evaluating the result for profitability. This is a rather large simplification, as "evaluating the result for profitability" includes simulating not just the transaction as-is in the mempool, but many variants of the transaction, all simulated as if sent through a rather large Solidity smart contract written up specifically for Tarkin. Here's a small subset of the actions taken during the post-new-tx-in-mempool hook:

 - Simulate the base transaction as if we sent it
 - Check the transaction for any ERC-20 token addresses in the hexdata
 - Simulate a bundled transaction where we approve any ERC-20s found to a set of known dexes
 - Find-and-replace any EOA addresses we find in hexdata with our own
 - Check any internal transaction recipients for dexes; approve tokens to those dexes as part of a multi-tx
 - Check ERC-20 balance differences as part of the simulated transaction; re-simulate with an attempt to sell said ERC20 at a known dex

Unfortunately, the possibility space for number of potential transactions to simulate here grows *incredibly* quickly. Tarkin can simulate and evaluate around 5,000 transactions per second, which is actually very good, but we have to be quite selective about *what* to simulate because the number of options grows exponentially with each criterion we add.

### Code

#### Solidity

All potential transactions are run through a [massive smart contract](https://polygonscan.com/address/0xF423b3D1e205b3A777D17262CDE89357583f30ef) I wrote to assist with simulation and execution of potentially profitable frontruns. This smart contract is about at the limit for smart contract size. It has methods that enable me to simulate transactions and return results (via staticcall), as well as revert-on-unprofitable transactino protection that will cause transactions to entirely fail if the frontrun was not successful, costing only the gas of a failed transaction - keeping the funds protected from bait transactions. Of course, this smart contract has 100% test coverage via an extensive set of both Forge/Foundry-based Solidity tests, as well as Typescript/Node based tests as well.

This smart contract not only simulates transactions and returns results, but scans for any ERC-20 tokens gained in the simulated transaction and returns these in a list, simulates potential profit from selling these tokens to any known dexes, and several other utility items that return information to the caller about the nature of the transaction. This information is, in turn, passed back to Tarkin.


#### Typescript

Tarkin's codebase is a standard Node project based in Typescript, with an executable runner and a few utility scripts for deploying the main Solidity smart contract onto various blockchains. It has integrated console output recording and hooks to report status updates to both Discord and Datadog.

The main loop listens for new transactions in the mempool, and, upon hearing them, simulates transactions as described above. It has an extensive set of hard-coded logic for what variants of the tx to attempt, such as "If I find an EOA address in the tx hexdata, simulate replacing that address with our own". Often, one transaction will be simulated with as much as 250 variants, if the transaction is complicated enough. 

Transaction selection to simulate is crucial; we prune transactions from EOAs to EOAs, transactions from known baiters, etc.

Tarkin connects to the blockchain node on the same machine via IPC - basically, an internal socket connection, which eliminates all the overhead of both TCP and HTTP, which would be a massive amount of data and slowdown if we connected that way.

Tarkin is multi-threaded, and spawns up to 8 sub-processes for simulation. Tarkin not only simulates new transactions, but old/stale ones in the mempool as well, since blockchain state may have evolved since the previuos non-profitable simulates of older, not-yet-executed transactions.

### Infrastructure / Devops

In order to be able to run simulations of potential transactions fast enough, I needed to run my own blockchain node for each chain that I wanted Tarkin to operate on. In order to be able to run massive amounts of simulations in parallel to explore the option space as efficiently as possible, I needed Tarkin to run on the same machine that the node was on. I parted together a server PC build, and ended up with a [Threadripper 2900](https://www.techpowerup.com/cpu-specs/ryzen-threadripper-2990wx.c2069)-based PC build with 256gb of RAM and, importantly, storage using 4x NVME hard drives in PCIE slots combined in a RAID (0). The 32 cores / 64 threads were needed to enable massive parallel compute; the RAM was needed to store the entire running node in memory (or as much as possible). The RAID0 storage using PCIE-based NVME SSDs was needed to enable as-fast-as-possible storage access; timing in this project was crucial.

Said server was hooked up to my home internet (Google Fiber), and at one point, Starlink.

At peak, I found I was able to run (via Docker instances) 5 dfferent nodes at the same time - ETH Mainnet + 4 sidechains (though, after about two hours, something would crash; I typically only ran Polygon & Binance Smart Chain regularly).

Monitoring was achieved via server outputs via webhook to my personal Discord server, as well as a Datadog dashboard that reported transactions, profits, errors, etc.
