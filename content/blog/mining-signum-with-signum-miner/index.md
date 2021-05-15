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
To start mining, the first thing you will need is some free disk space. This is most easily accomplished by using your existing free space or, as I like to do, find or buy some old hard drives and plug them into the computer.

This article isn't a tutorial on how to do all the Linux commands required to format, mount, and use a hard disk. Google things like 'linux format hard drive' and 'linux mount partition' for help with those things. What you need to know is:

  * where your partition is mounted
    * example: `/mnt/burst-hd1`
  * that you should format the partition with a filesystem that supports multi-terabyte files, like the `ext4` filesystem

Once you've got all that done, it's time to get started.

# Every ~~Story~~ Miner Needs a Plot
Just like every story needs a plot, so too does every Burstcoin miner. Well, at least one plot. Unlike telling a story, in Burstcoin you can have as many plots as you want. At this point you might be wondering: What exactly is a plot?

In Burstcoin, a plot file is a collection of cryptographic hashes stored in a specific structure on your disk. I won't go into the nitty-gritty details of how it's structured or the details of creating the hashes. You only need to know that these about plot files:
  * they are cryptographically linked to your Burstcoin account
  * they can be any size from small to excessively large
  * they can be stored anywhere as long as your miner software can see them

Creating a plot file is pretty simple, but it can be very time consuming. You're going to need a tool called engraver. [Download it here][engraver-download] and extract it to your preferred working folder. (`tar -xf engraver-2.4.0-x86_64-unknown-linux-gnu-cpu-gpu.tar.xz`)

Change directory to where you extracted the engraver binaries (open a terminal if you're working in a Linux GUI). There should be two files: `engraver_cpu` and `engraver_gpu`. You can choose either of them but this article will focus on the CPU version.

Make the `engraver_cpu` file executable by typing `chmod +x engraver_cpu`. This is the program that will create the plot files used for mining later. Before engraving starts, we need to collect some information:

| Information              | Where to get it                                                                                                                                                                                                      |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Numeric account ID       | * BRS Wallet - On the top left of the window, click the `Copy Numeric Account ID`<br />* Phoenix Wallet - On the top left, click the little drop-down arrow below your Burstcoin address and click `Copy Account ID` |
| Free disk space in kb    | Type `df -BK /mnt/burst-disk`, changing the path to match your own desired plot file location. Record the amount under the `Available` column.                                                                       |
| Number of nonces to plot | Divide the free space in kb by 256. Example: `1367753384 kb / 256 = 5342786 nonces`                                                                                                                                  |

<div class="note">Note: If you don't intend to use the entire drive, record the free disk space as the amount you wish to use in kilobytes.</div>

Now that you have the required information, it's time to start engraving your plot files. Start by creating a folder on your drive where you want to store the plots. It's suggested to store them in a folder rather than on the root of the drive.

```bash
mkdir /mnt/burst-disk/plots
```

Next, it's time to launch engraver:

```bash
./engraver_cpu -s <start nonce> -n <total nonces> -p /mnt/burst-disk/plots
```

`<start nonces>` should be replaced by the nonce you want to start at. You should never overlap nonces with other plot files you have created, so my preferred method is to keep track of my plotted nonces in a spreadsheet, and add one to my highest nonce to choose my next starting nonce. For the first plot, you can use `0`.

`<total nonces>` should be replaced by the 'number of nonces to plot' in your collected information above. Add this number to the start nonce to determine the 'ending nonce' of this plot file.

You should also replace the path to match your own desired plot files location.

Once you execute the command, engraver will start creating your plot files. This is going to take a long time if you have a lot of space to plot. It will block your terminal, so if you aren't using a GUI, consider running it in 'screen', 'tmux', or 'byobu'. These are all command line multiplexers. Google them to learn more about them.

Wait until the plotting is complete, then move on to mining.

<div class="note">Note: If your engraver dies or you need to quit it early, worry not! Engraver is smart enough to pick up where it left off. To get this behavior, however, you DO need to type the exact same command you did the first time. If you make a mistake in the numbers or ID, it'll start a brand new plot file.</div>

# Get Those Scavenging Gloves On




[discord-join-link]: https://discord.gg/aBFeCNPgQd "Official Burstcoin Discord"
[engraver-download]: https://github.com/PoC-Consortium/engraver/releases/latest "Latest engraver download"
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
