# Ansible

Making life simple with Ansible, we will use it to configure our nodes within the Kubernetes cluster.

## Prerequisites

We are going to use our laptop as the base workstation to manage all cluster nodes, including the master node.

> \[!NOTE]\
> I am using just one master and one worker node here. You can add as many more as you like/have the budget for!

First, confirm all nodes are up:

```bash
ping master.local # OR 10.0.0.98 in case you know the IP address
ping node01.local # OR 10.0.0.131 in case you know the IP address
```

Before we can work with ansible, there's some manual configuration to be done. Edit the `/etc/hosts` file on your workstation with root privileges:

```bash
sudo vi /etc/hosts
```

Add the following lines to it:

```txt
10.0.0.98       master master.local
10.0.0.131      node01 node01.local
```

With this in place, you can now access your nodes using `master`, `master.local`, `node01` and `node01.local` names. Feel free to add more in case you have more number of nodes.

### PKI Configuration

- Dump/copy the public key generated when flashing the OS to the microSD cards to all the other servers to which the main ansible machine is going to talk to:

  ```bash
  ssh-copy-id -i ~/.ssh/k3s.pub master
  ssh-copy-id -i ~/.ssh/k3s.pub node01
  ```

Here are some commands in case you want to troubleshoot the PKI environment:

- To list all active keys:

  ```bash
  ssh-add -L
  ```

- Remove an ssh-key:

  ```bash
  ssh-add -D <key-name>
  ```

## Working with Ansible

- First things first, we need to install ansible on our workstation:

  ```bash
  brew install ansible
  # ansible --version : to confirm ansible is installed successfully
  ```

- Create an `inventory` file (this replaces /etc/ansible/hosts):

  ```ini
  [control]
  master ansible_connection=ssh

  [workers]
  node01 ansible_connection=ssh

  [homelab:children]
  control
  workers
  ```

- Create a configuration file `ansible.cfg` (overwrites the default configuration options):

  ```ini
  [defaults]
  inventory = inventory
  private_key_file = ~/.ssh/k3s
  ```

Alright, we are all set to use ansible! More on that in the next part.

Before we move on, let's make sure our ansible workstation is configured correctly and can connect to the nodes on the network successfully

```bash
# We are going to execute ping with ansible, the "homelab" is a group we specified in the "inventory" file.
# Instead of specifying the inventory file with the -i flag, we have it configured in "ansible.cfg".
# The -m flag specifies an ansible module, and here we're going to use the "ping" module
ansible homelab -m ping
```

The result should be:

```bash
master | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
node01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

Of course, if you have more worker nodes connected to the network, the output will show the ping output to those nodes as well.

[Prev: Installing the OS](./02_base_os.md) | [Next: Install K3s](./04_k3s_install.md)
