# Storage setup

For storage purposes, we will be using `Longhorn` storage driver, which is a native Kubernetes storage software with support for `arm64` architectures, so this works perfectly fine with our Raspberry Pi 4 nodes. Since we disabled the `local-storage` installation on our Kubernetes cluster, this is our go-to storage option.

## Longhorn

To use `Longhorn` to setup storage volumes, all we need to do is mount the disk under `/var/lib/longhorn`, which is the default data path. I changed it to `/storage01` since it is easier to find and when adding more disks, it is easier to add under `/storage02,03,04...`.

> \[WARNING]\
> There is an issue with Raspberry Pi: the names for disks are assigned almost at random. So even if you have USB disks in the same slots on multiple nodes, they can be named at random: `/dev/sda` or `/dev/sdb`, and there is no easy way of enforcing the naming.

## Software requirements

Install the following software on all nodes, run the following commands from your ansible workstation:

```bash
ansible homelab -b -m apt -a "name=nfs-common state=present"
ansible homelab -b -m apt -a "name=open-iscsi state=present"
ansible homelab -b -m apt -a "name=util-linux state=present"
```

## Identify disks for storage

Since we will be using ansible, we will add on to the `ansible/inventory` file a few variables that will help us with mounting multiple storage disks to the Raspberry Pis with ease.

Let's use the `lsblk -f` command on every node to look for disk labels:

```bash
# cd ansible/
ansible homelab -b -m shell -a "lsblk -f"
```

Each node has two disks, `sda` and `sdb`, but assigned at boot. Every disk that splits to `<name>1` and `<name>2`, and has `/boot` and `/` as mount points, these are our OS disks. The other ones are our storage drives.

Update the `inventory` file for ansible with the `var_disk` variable to point to the storage drive.

Here's what my `inventory` file looks like (I have one disk which is mounted to `sda1`):

```ini
[control]
master ansible_connection=ssh var_hostname=master

[workers]
node01 ansible_connection=ssh var_hostname=node01 var_disk=sda1

[homelab:children]
control
workers
```

### Wipe your disks

Before using the storage drives, it's usually a good practice to format it to `ext4` file system format that works best with Linux. Let's do it using ansible:

```bash
#wipe
ansible workers -b -m shell -a "wipefs -a /dev/{{ var_disk }}"
#format to ext4
ansible workers -b -m filesystem -a "fstype=ext4 dev=/dev/{{ var_disk }}"
```

This will clear the partition table and format the disk to `ext4` format.

### File system and mount

We need to get unique identification number of our storage disks so we can mount them every time, even if the `/dev/...` label changes for them. For that, we need the UUID of the disks.

```bash
ansible workers -b -m shell -a "blkid -s UUID -o value /dev/{{ var_disk }}"
```

If you have multiple nodes with multiple disks, add these UUID values to your `inventory` file for ansible:

```ini
[control]
master ansible_connection=ssh var_hostname=master

[workers]
node01 ansible_connection=ssh var_hostname=node01 var_disk=sda1 var_uuid=d7bbc71e-4a42-a944-8f30-e7436e24397d

[homelab:children]
control
workers
```

Using ansible, let's mount the disk to /storage01

```bash
ansible workers -m ansible.posix.mount -a "path=/storage01 src=UUID={{ var_uuid }} fstype=ext4 state=mounted" -b
```

## Mount storage disks manually

> \[WARNING]\
> It is recommended you use Ansible to format and mount disks to your nodes, since having multiple nodes with multiple storage disks will only make mounting these disks manually very time consuming.
> If you have smaller number of nodes and disks, run all commands mentioned in this guide as the `root` user.

To mount an external storage drive, follow the instructions mentioned below:

- Identify the disk you want to mount:

  ```bash
  df -h
  # OR
  # lsblk -f
  ```

  This will list all the connected device storage:

  ```txt
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/root        29G  3.1G   25G  12% /
  devtmpfs        1.8G     0  1.8G   0% /dev
  tmpfs           2.0G  8.0K  2.0G   1% /dev/shm
  tmpfs           2.0G  8.6M  1.9G   1% /run
  tmpfs           5.0M  4.0K  5.0M   1% /run/lock
  tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
  /dev/mmcblk0p1  253M   53M  200M  21% /boot
  tmpfs           391M     0  391M   0% /run/user/1000
  /dev/sda1       932G  595G  337G  64% /media/pi/My Passport
  ```

- With the result from above, identify the drive you want to mount:

  ```txt
  /dev/sda1       932G  595G  337G  64% /media/pi/My Passport
  ```

- Retrieve the disk UUID and TYPE

  ```bash
  sudo blkid /dev/sda1
  ```

  This will generate an output as shown below:

  ```txt
  /dev/sda1: LABEL="My Passport" UUID="8A2CF4F62CF4DE5F" TYPE="ntfs" PTTYPE="atari" PARTUUID="00042ada-01"
  ```

  > NOTE: Depending on the type of filesystem, you may have to install additional drivers. See [here](https://pimylifeup.com/raspberry-pi-mount-usb-drive/) for more details.

- Next, we will mount the drive to the raspberry-pi:

  ```bash
  # create the mount directory
  sudo mkdir -p /storage01
  # update permissions - not required if running k3s as root user
  sudo chown -R sid:sid /storage01
  ```

  ```bash
  # mount the drive to the directory
  sudo mount /dev/sda1 /storage01
  ```

- Update the `fstab` file:

  ```bash
  sudo nano /etc/fstab
  ```

  Add the following to the bottom of the file with the correct `UUID` and `TYPE` from step 3.

  ```txt
  UUID=[UUID] /storage01 [TYPE] defaults,auto,users,rw,nofail,noatime 0 0
  ```

To make sure the drives are restored after the Pi has been shut down then run the following command:

```bash
sudo shutdown -r now
```

## Install Longhorn

We will use the Longhorn helm chart to install Longhorn storage driver:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --set persistence.defaultClassReplicaCount=1 --set defaultSettings.defaultDataPath="/storage01" --set service.ui.loadBalancerIP="10.0.0.201" --set service.ui.type="LoadBalancer"
```

Give it some time, some PODs will restart, but in the end, everything under the `longhorn-system` namespace should be `1/1 Running` or `2/2 Running`.

[Prev: Network Configuration](./05_network.md) | [Next: Traefik Ingress](./07_traefik.md)
