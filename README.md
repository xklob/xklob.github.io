# Selected Projects

## Tarkin - Generalized Frontrunner MEV Bot

Inspired by the Paradigm [Dark Forest paper](https://www.paradigm.xyz/2020/08/ethereum-is-a-dark-forest) published in late 2020, I set out to build the so-called "cosmic horror" named in the paper - a generalized frontrunner. In the words of Dan Robinson, this bot would "look for any transaction that they could profitably frontrun by copying it and replacing addresses with their own" in order to create money out of thin air. After several frustrating iterations, numerous roadblocks, and much learning, I arrived at the first *working* version on the third attempt: [Tarkin](https://en.wikipedia.org/wiki/Grand_Moff_Tarkin).

### Capabilities

todo

### Infrastructure / Devops

In order to be able to run simulations of potential transactions fast enough, I needed to run my own blockchain node for each chain that I wanted Tarkin to operate on. In order to be able to run massive amounts of simulations in parallel to explore the option space as efficiently as possible, I needed Tarkin to run on the same machine that the node was on. I parted together a server PC build, and ended up with a [Threadripper 2900](https://www.techpowerup.com/cpu-specs/ryzen-threadripper-2990wx.c2069)-based PC build with 256gb of RAM and, importantly, storage using 4x NVME hard drives in PCIE slots combined in a RAID (0). The 32 cores / 64 threads were needed to enable massive parallel compute; the RAM was needed to store the entire running node in memory (or as much as possible). The RAID0 storage using PCIE-based NVME SSDs was needed to enable as-fast-as-possible storage access; timing in this project was crucial.

Said server was hooked up to my home internet (Google Fiber), and at one point, Starlink.

At peak, I found I was able to run (via Docker instances) 5 dfferent nodes at the same time - ETH Mainnet + 4 sidechains (though, after about two hours, something would crash; I typically only ran Polygon & Binance Smart Chain regularly).
