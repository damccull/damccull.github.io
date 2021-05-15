+++
title = "Mining Signum with the Signum-Miner Tool"
date = 2021-05-01T14:06:00Z

description = "Learn to makes mining plots and mine Signum on them manually."
draft = true
[taxonomies]
tags = ["Signum", "Proof of Capacity"]
categories = ["CryptoCurrency"]
+++

# So You Want to Mine Burstcoins?
That's a great idea! The Burstcoin network needs to grow and become even more decentralized than it already is. The good news is that mining is super simple with the BTDEX app. It functions as a Burstcoin wallet; a decentralized, cross-chain exchange; and a plotter/miner. With this one tool, you can do it all. If that's all you want, read _[Mining with your Hard Drive in 2021][jjos-article]_ by one of the current Burstcoin developers, and then [go check out the BTDEX website][btdex]. While you're at it, [please click here to join the Burstcoin community on Discord][discord-join-link].

BTDEX is a great choice to get started, but eventually most people want to stop running their critical mining infrastructure on their every day use PC. This article is about setting up a headless miner under any Linux distribution.


# Setting Up the Disks
You need to format

# Every ~~Story~~ Miner Needs a Plot
Just like every story needs a plot, so too does every Burstcoin miner. Well, at least one plot. Unlike telling a story, in Burstcoin you can have as many plots as you want. At this point you might be wondering: What exactly is a plot?

In Burstcoin, a plot file is a collection of cryptographic hashes stored in a specific structure on your disk. I won't go into the nitty-gritty details of how it's structured or the details of creating the hashes. You only need to know that these about plot files:
  * they are cryptographically linked to your Burstcoin account
  * they can be any size from small to excessively large
  * they can be stored anywhere as long as your miner software can see them

Creating a plot file is pretty simple.


[discord-join-link]: https://discord.gg/aBFeCNPgQd "Official Burstcoin Discord"
[engrager-download]: https://github.com/PoC-Consortium/engraver/releases/latest "Latest engraver download"
[scavenger-download]: https://github.com/PoC-Consortium/scavenger/releases/latest "Latest scavenger download"
[btdex]: https://btdex.trade/ "BTDEX"
[jjos-article]: https://jjos2372.medium.com/mining-with-your-hard-drive-in-2021-19d9f4a1368 "Mining with your Hard Drive in 2021"


[signum-download]: https://github.com/signum-network/signum-node/releases/latest "Signum Downloads"
[jportable]: https://portableapps.com/apps/utilities/java_portable "jPortable"
[docker-install-instructions]: https://docs.docker.com/install/ "Docker Install Instructions"
[docker-install-instructions-ubuntu]: https://docs.docker.com/install/linux/docker-ce/ubuntu/ "Docker Install Instructions for Ubuntu"
[docker-get-started]: https://docs.docker.com/get-started/ "Docker's Get Started Tutorial"
[docker-compose-install]: https://docs.docker.com/compose/install/#install-compose-on-linux-systems "Install docker-compose on Linux"
[discord-join-link]: https://discord.gg/eVFRx7DECX "Official Signum Discord"
