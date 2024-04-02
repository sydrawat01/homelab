# Hardware

## Raspberry Pi 4 K3s cluster

This is the list of hardware I'm using. To be honest, this is a "junkyard" buikd, you can achieve the same with more and better hardware, but this is what I have.
The minimal but ideal setup I would go for is 3x Raspberry Pi with USB Disks (SSD if possible). This is not a HA setup, since I am working with limited supplies and on a strict budget.

> \[!IMPORTANT]\
> The softwares we will be installing and using are amd64 compatible, so if you're using an older revision of the Raspberry Pi like 3 or lower, you will run into some issues where the software will either not install or not run, citing compatibility issues.
> Make sure you are using Raspberry Pi 4 and above for all intents and purposes going forward.

- **1x Raspberry Pi 4** (8GB RAM for the master node)
- **1x Raspberry Pi 4** (2GB RAM for the worker node, this one is too slow, I'm thinking of replacing it with 2x Pis with 8GB RAM)
- **1x 16GB Memory Card**(Samsung boot disk for master node)
- **1x 8GB Memory Card**(SanDisk boot disk for worker node)
- **1x 64GB USB Disk** (SanDisk USB 3.2 Gen1 64GB)
- **1x 1TB USB Disk** (Seagate USB HDD 3.1 1TB)
- **2x Power Supply 5V 3A (15W)** (USB Power sockets)
- **2x USB-C charging cables** (USB-C Power cables)
- **2x RJ45 network cables** (Ethernet cables which connect to the home router)

Currently I am using the 64GB thumb drive as storage on the master node and the 1TB HDD on the worker node for storage. I plan to use the 64GB thumb drives as a boot disk for the nodes. Memory cards burn out sooner or later, so this is a better and safer option. Again, if you are short on resources, I suggest go with the basic microSD card approach to flash your Raspberry Pi 4.

## Speed Benchmarks

I haven't performed any storage speed benchmarks since this is a "junkyard" build, but if you're interested, here's a script you can use to test the speed benchmarks: [https://github.com/TheRemote/PiBenchmarks](https://github.com/TheRemote/PiBenchmarks)

[Next: Design Goals](./01_design_goals.md)
