# Operating System

I have chosen the headless Raspberry Pi OS Lite (64-bit) which is a port of the Debian Bookworm with no desktop environment.

Flashing the OS onto the microSD cards is straight-forward. Just download and install the Raspberry Pi Imager [link](https://www.raspberrypi.org/downloads/) to flash the cards with the appropriate OS of your choice.

If you're on a MacBook like me and have `homebrew` installed, use the following command to install the Raspberry Pi Imager:

```bash
brew install --cask raspberry-pi-imager
```

## PKI (Public Key Infrastructure)

We need to configure the SSH keys on our nodes. We will do this from our workstation and copy them to the nodes.

> \[!NOTE]\
> Remember to run the ssh-keygen commands as root user, so we append `sudo` to the ssh-keygen commands below.

- Generate ssh key-pair

  ```bash
  ssh-keygen -t ed25519 -C "homelab"
  # follow the on-screen instructions: save the key to ~/.ssh/k3s
  ```

- This will generate a public and a private key-pair with the name `k3s` and `k3s.pub`.

- Add the generated ssh-keys to `known_hosts`

  ```bash
  ssh-add k3s
  ```

## Flash the OS

[TODO]: Show steps indicating how to install and flash the OS to the microSD cards using Raspberry Pi Imager.

## Initial setup

Once you have successfully installed the OS on the microSD cards, we need to configure a few things in order to run the k3s kubernetes distribution and for the underlying container runtime (`containerd`). This will save us a reboot later on.

### One-time configuration

Let's configure the `cmdline.txt` file, do not replace the whole line in the file, just append the values mentioned:

If you're using the `Debian Bookworm` Raspberry Pi port like me, you can find the `cmdline.txt` file in the `/boot/firmware/` folder. Navigate to this folder and edit the `cmdline.txt` file with the following values:

- `group_enable=cpuset`
- `cgroup_enable=memory`
- `cgroup_memory=1`

Once done, plug-in the microSD cards and power up the Raspberry Pis with the network cables hooked-up to the router. Wait for the Pi's to boot up and then login to the Pis to disable swap memory.

I found an excellent blog for disabling the swap for Raspberry Pi SD cards ([link](https://www.dzombak.com/blog/2023/12/Stop-using-the-Raspberry-Pi-s-SD-card-for-swap.html)).

To completely disable swap on Raspberry Pi OS:

```bash
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo update-rc.d dphys-swapfile remove
```

If you’re certain you won’t want to reenable swap in the near future, you can completely remove the `dphys-swapfile` manager with:

```bash
sudo apt purge dphys-swapfile -y
```

[Prev: Design Goals](./01_design_goals.md) | [Next: Configure Ansible](./03_ansible.md)
