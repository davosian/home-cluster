# Getting Started

## Intended Audience

It is a fine line to walk on to provide a "simple" guide for a complex mix of technologies. While I am trying my best to guide you through the whole process, I am assuming that you know your way around [git](https://git-scm.com/doc), [Ansible](https://docs.ansible.com/ansible/latest/index.html) and the [terminal](https://www.digitalocean.com/community/tutorials/an-introduction-to-the-linux-terminal). Furthermore, I assume you have some basic knowledge around [Kubernetes](https://kubernetes.io/docs/home/) and [K3s](https://rancher.com/docs/k3s/latest/en/). This is not a tutorial for any of these technologies but rather a guide to getting started with a home lab running Kubernetes in an opinionated fashion.

## Introduction

You want to set up a brand new cluster from scratch using this repository? Here are the first steps to get you from zero to a running k3s cluster in your home lab:

## Cluster Management Environment

First you need a computer which will be used to set up the cluster. Ideally, your computer should run Linux or macOS. The following instructions have been tested with a MacBook Pro running macOS Big Sur 11.2.1.

First you need a few prerequisites:

```sh
# Install homebrew from https://brew.sh/
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Task from https://taskfile.dev/#/installation
brew install go-task/tap/go-task

# Install Ansible
brew install ansible
```

Now make sure you are in the root directory of this repository before you execute the following commands:

```
# Local OS Dependencies
task deps:install:dev

# Ansible Dependencies
task ansible:deps
```

The next step depends on the shell you are using. If you are using Bash, open `~/.profile`, if you are using ZSH, open `~/.zshrc`. Append the following to the end:

```
# make sure the ansible binary is found
export PATH="~/Library/Python/3.9/bin:$PATH"

# direnv support
# https://direnv.net/docs/hook.html
eval "$(direnv hook zsh)"
```

Now open a new shell in the project root directory before you continue to make sure your new settings take effect.

If you are getting an error like this one:

```
direnv: error /Users/yourname/home-cluster/.envrc is blocked. Run `direnv allow` to approve its content
```

do as instructed:

```
direnv allow
```

Now verify that some necessary variables are set:

```
echo $ANSIBLE_CONFIG
echo $KUBECONFIG
```

Both commands should return a path inside your repository. This is important for some other tooling to work properly later on.

## Cluster Hardware Setup

This guide assumes you have 3 Intel NUC devices you want to connect to a k3s cluster. Why NUCs? Well, we have to start somewhere and NUCs are a popular, affordable choice for home labs. Feel free to use any hardware available to you and adapt the instructions accordingly.

Also, it is assumed that the NUCs only have internal NVMe disks and no SSD attached. Is yours is different, check the `storage` section for `worker-nodes.yaml` and `master-nodes.yaml`. The SSD typically resides unter `/dev/nvme0n1` while the NVMe disk gets mounted at `/dev/sda`. Since this is an opinionated `Getting Started` guide, we will not go into the details to configure the system differently.

In addition, this guide assumes that you also have a NAS to be using for persistent storage.

### BIOS Upgrade

First, make sure your BIOS is up to date by checking the current firmware and comparing it with Intel's latest: connect mouse, keyboard and a monitor to your NUC and hit `F2` when you see the "Intel" startup screen to enter the BIOS setup. Check the firmware version and compare it to the latest one available for your model: https://downloadcenter.intel.com/product/70407/Intel-NUC-Kits. If there is a newer one available, download it onto a FAT32 formatted USB stick and follow the instructions to upgrade the BIOS.

Repeat this process for all your NUCs.

### Boot order priority

We want to be able to do a fresh install on the NUCs without having to attach keyboard, mouse and monitor in the future. While not being the most secure setup, it allows for easy experimentation and handling of the system. See the "Starting Over" chapter in this guide for more details.

Boot into the BIOS again using `F2`. Go to the `Boot Order > Advanced > Boot Configuration > UEFI Boot` settings and check the option `Boot USB Devices First`. Also make sure that `Boot Order > Advanced > Secure Boot > Secure Boot` is disabled. Confirm the new settings with `F10 > Yes`.

Repeat this process for all your NUCs.

### Prepare the base OS

This `Getting Started` guide assumes, you want to have one master node in your cluster (carrying the control plane) and two worker nodes ("only" serving your containers and providing persistent storage).

Open a terminal and navigate to the directory `integrations > ubuntu-autoinstall` of this git repository.

`master-nodes.yaml` is used to configure the base OS for the master node, `worker-nodes.yaml` sets up the worker nodes.

Note: if you do not use NUCs for your hardware, it is likely that you need to figure out the exact configuration for your hardware. The easiest way is to first install Ubuntu manually on those machines by following the official instructions, then copying the configuration from the new system onto your host through `scp`. You will find the configuration under `/var/log/installer/autoinstall-user-data`:

```
scp ubuntu@my-new-ubuntu-system:/var/log/installer/autoinstall-user-data .
```

Now merge this configuration into `master-nodes.yaml` and `worker-nodes.yaml`.

If you do use NUCs, you can most likely use the default configuration from this repository:

In both files, `master-nodes.yaml` and `worker-nodes.yaml`, go the section `autoinstall > ssh > authorized-keys`. Replace the entries for the public keys with the public keys of machines you want to use to access the NUCs by ssh. Typically you find those keys inside of `less ~/.ssh/*.pub`. If you have multiple files in this directory, you can navigate them with `:n` and to go back, use `:p`.

Also, inside of `master-nodes.yaml`, for `autoinstall > storage > config > id: root-disk-0 > path`, you should enter `/dev/nvme0n1` if you have installed both a NVMe disk and an SSD disk inside your NUC. This will make your SSD the primary disk.

For `worker-nodes.yaml`, you should uncomment the last section `# Block device for distributed storage` if your NUC has an SSD built in in addition to the NVMe disk and you want to use it for storage.

Now we can prepare the `iso` images that we need to load onto the NUCs:

Make sure that Docker is running on your local computer (cluster management environment). Then execute the following commands to create the iso files:

```
# Build the docker image to generate the ISOs
docker build -t autoinstall-ubuntu:latest .

# Generate the ISOs
docker run --rm -v "$PWD"/build:/build autoinstall-ubuntu
```

As a last step, we have to load the iso files onto a USB stick:

Insert the USB stick into your computer.

Open `balenaEtcher.app` and select `Flash from file`. Navigate to the newly created iso image called `integrations > ubuntu-autoinstall > build > ubuntu-20.04.2-live-server-amd64-autoinstall-masters.iso`. Under `Select target`, check the box next to the USB stick you want to use.

Note: if you want to prepare the worker nodes, use the other iso image instead: `ubuntu-20.04.2-live-server-amd64-autoinstall-workers.iso`. All remaining steps will be identical.

Careful: you are going to override the content of the complete stick!

Then click on `Flash!` and wait for the process to complete. After receiving a success message from balenaEtcher, you can remove the stick from your computer.


### Installing the base OS

Note that you should be able to follow the steps outlined in this section without having to attach a keyboard, mouse and monitor. You will know that it worked out when the NUC turns off itself after about 5 minutes. If you are doing this for the first time however, I would still recommend to add at least add a monitor so that you can follow along and learn about possible issues.

Insert the USB stick into the NUC, then turn it on. The NUC should now boot up from the USB stick, install Ubuntu 20.04.2 and shut down again. Once it has shut down, remove the USB stick and turn it on again.

Note that all NUCs that get this installation will end up with the same host name. This collition should get remedied immediately by performing the configuration outlined in the next section.


### Get your NUC a fixed IP address

To make it easy for you to find the NUC in your network later again, go to the web interface of your router (typically at `http://192.168.1.1:8443 for Unify routers) and assign a fixed IP address to your NUC. Now turn off the NUC by a quick press on its power button. Push the button to turn on the NUC again. This will reboot it and should assign the new IP address.

Verify that after rebooting, the NUC now carries the statically assigned IP address. Take note of that address. You will need it later in chapter `abc`.


### Configuring the base OS

Once we have a clean setup with Ubuntu installed, it is time to optimize the system for our use case. To do this, we use an ansible playbook.

Navigate inside the `ansible > inventory > k3s-single-control-plane` directory and open up the `hosts.yml` file.

Under `all > children > control-nodes > hosts`, enter the hostname you would like to use for your master node, e.g. `nuc-master`.

Under `all > children > generic-nodes > hosts`, enter the hostnames you would like to use for your two worker nodes, e.g. `nuc-worker-1` and `nuc-worker-2`. Make sure you have matching filenames in `ansible > inventory > k3s-single-control-plane`: `nuc-master.yml`, `nuc-worker-1.yml` and `nuc-worker-2.yml`.

For the workers, only enable `rook-ceph` if you have an additional disk inside and you set it up in the `Prepare the base OS` section. Otherwise, set it to `enabled: false`.

Now open each one of these three files and make sure to set the IP address to the static one you have picked under `Get your NUC a fixed IP address`. Also make sure to have uncommented the line `ansible_become_pass: "ubuntu"`.

Next, you might want to adjust `ansible > inventory > k3s-single-control-plane > group_vars > all > ubuntu-settings.yml`: set the time zone for your region, ntp servers for your region and optionally add additional public ssh keys if you have not yet done so when you created the Ubuntu iso above.

It is time to apply the configuration to your NUCs now. Make sure all of them are running and reachable under their respective IP address you have configured above.

```
ansible-playbook playbooks/ubuntu/prepare.yml -i inventory/k3s-single-control-plane/hosts.yml
```

When asked whether you want to reboot the nodes when the playbook completes, select `Y`.

Now you can go ahead and comment out the line `ansible_become_pass: "ubuntu"` because you can now connect with the certificates you have provided in the setup.

## Storage Setup

Rook-Ceph
https://github.com/democratic-csi/democratic-csi
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

## K3s Setup

### Installing K3s

```
# make sure you are inside the ansible directory
cd ansible/

# install k3s on all hosts using ansible
ansible-playbook playbooks/k3s/install.yml -i inventory/k3s-single-control-plane/hosts.yml
```

Confirm with `Y` when asked whether you want to proceed with the installation and verify that the steps have completed without errors.

In order to interact with your cluster, you need to make sure that environment variables are set:

```
# make sure you are inside this repo's root directory
cd ..

# install k3s on all hosts using ansible
ansible-playbook playbooks/k3s/install.yml -i inventory/k3s-single-control-plane/hosts.yml

export KUBECONFIG=...../home-cluster/kubeconfig    
```


### Configuring K3s



### Reinstalling K3s


## Where to go next?


## Starting Over

Boot from USB...

## Upgrading the OS

Ubuntu has been set up to automatically install security fixes. 

In addition it is sometimes a good idea to upgrading additional components as well which might require a reboot. For this you can use another

## Testing without Hardware

If you want to test your setup first before modifying your physical cluster, you can use virtual machines instead of NUCs. This way you can easily experiment and go back to a previous state by leveraging snapshots.

### Setting up Vagrant

### Preparing a virtual cluster