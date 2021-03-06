# ansible_tutorial with nested LXC containers

Started following the ['Ansible' tutorial playlist](https://youtube.com/playlist?list=PLT98CRl2KxKG7LKdWeXYUe6_UTeUybE2Z)  on the [LearnLinuxTV](https://www.youtube.com/c/LearnLinuxtv) YouTube Channel. There was a bit of strangeness in the channel's author's sources at about episode 13. Jumped past roles and host-variables to templates. I'm going to refactor as best I can.



## The Goal



Provide a well-worked example using ansible to configure a test environment implemented on nested lxc containers. With an aim to making it easy to provision development environments on all sorts of platforms.

I am interested in being able to place a PyTorch environment quickly on my workstation, and  any cloud-server providing affordable machine-leaning GPU.

### Looks like

1. Generate a working development environment with an lxc Ubuntu container. 
2. Within the host container provision nested containers from several distributions - Ubuntu, Debian, CentOS.
3. Create and deploy a single ssh key, and ansible user for ansible to manage the nested containers.
4. Using ansible, configure those nested containers in groups: WebServers, FileServers, DatabaseServers

The various distributions will promote some cross-platform challenges...

### Try to

- [Don't Repeat Yourself ](https://wiki.c2.com/?DontRepeatYourself)  (DRY)
- Make it readable
- Be an example for learning
- Improve my use of GitHub and BitBucket 

### Host Container

- A main, current release, [Ubuntu Focal Fossa LTS](https://releases.ubuntu.com/20.04/) container, which can be logged into using an ssh key, with password login disabled. It will still be accessible for alternate login via the `lxc exec` interface.
-  A reasonable development environment [neovim](https://neovim.io/), [Zsh](https://en.wikipedia.org/wiki/Z_shell), [Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh), [zplug](https://github.com/zplug/zplug) plugins for history and navigation.
- A test environment managed from a git repository
- X11 support. See:  [Running X11 software in LXD containers](https://blog.simos.info/running-x11-software-in-lxd-containers/)









-------------------

## Below are pre-refactoring notes...

### Create required lxc profile

You will need to substitute in your public key you made for [Ansible](https://en.wikipedia.org/wiki/Ansible_(software)) by editing 

```
lxc profile create ansible
cat lxc-profiles/ansible-base-profile.yml | lxc profile edit ansible by editing `ansible-base-profile.yml`
```

### create your host container

```
lxc launch ubuntu:20.04 --profile default --profile ansible ansible-host

```


Within the Ubuntu 20.04 [LTS](https://ubuntu.com/blog/what-is-an-ubuntu-lts-release) [LXC](https://ubuntu.com/server/docs/containers-lxc) container:

1. Install [miniconda]() using `install-miniconda.sh`
2. `apt purge ansible` - to make sure there is no ansible installation based on a python which conflicts with 
the miniconda environment
3. `conda install -c conda-forge mamba` -  [mamba](https://github.com/mamba-org/mamba) is a much quicker tool for building [conda](https://towardsdatascience.com/environment-management-with-conda-python-2-3-b9961a8a5097) [python virtual environments](https://docs.python-guide.org/dev/virtualenvs/)
4. `mamba env create -f ./environment.yml` - to create a conda environment with ansible installed using a [yaml environment specification file](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#creating-an-environment-with-commands).
5. Add the [incantation to activate](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment) the environment to your user's shell startup script `echo 'conda activate ansible_env >>~/.bashrc'` change if you are using [zsh](https://medium.com/@harrison.miller13_28580/bash-vs-z-shell-a-tale-of-two-command-line-shells-c65bb66e4658) to ~/.zshrc

### create the target containers with the ansible host container

```
export ubuntu_containers=("one" "three" "two" "ws")
export centos_containers=("centos")
export target_containers=("one" "three" "two" "ws" centos)

for i in ${ubuntu_containers}; 
do 
    lxc launch ubuntu-minimal:focal --profile default ${i}
done

for i in ${centos_containers}; 
do 
    lxc launch images:centos/8 --profile default
done

# some other stuff... TBD
```

## Lesson Notes

### Getting started with Ansible 07 - The 'when' Conditional

The CentOS image `images:centos/8` is missing a lot of things needed to follow the tutorial

- packages: `openssh-server sudo firewalld`
- configuration: an ansible sudoer user, firewall configuration for httpd

[How To Install the Apache Web Server on CentOS 8](https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-centos-8) - I followed these ntructions, but had to add http service because could not get connection without https certificate. So I used the alternate firewall setup from here:
[How to Install Apache on CentOS 7](https://www.liquidweb.com/kb/how-to-install-apache-on-centos-7/)

### Getting started with Ansible 10 - Tags

Running a playbook with a list of tags causes only plays or tasks which  have all the tags to be executed. The list is AND. See [Selecting or skipping tags when you run a playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tags.html#selecting-or-skipping-tags-when-you-run-a-playbook)

```
ansible-playbook site.yml --tags "apache,db"
```

## General Notes

### Tools

- [httpie](https://github.com/httpie/httpie#about-this-document) very useful commandline http client for testing.

### zsh aliases

```
# capture your containers' ip addresses in a handy shell variable for iterating over

export servers_ip=$(lxc ls -c 4 --format csv| cut -d" " -f1  | xargs) 


#Using it here to copy public keys to ubuntu servers - probably will not work for CentOS

for i in ${servers_ip}; do ssh-copy-id -i ~/.ssh/ansible ubuntu@$i ; done

```

### Iterate over containers in parallel

```
# Use those cores!
 % for i in $servers; do lxc restore ${i} base &; done && wait
```

### An example ssh_config file to manage host names and public keys

Makes it easy to work with multiple keys and ansible.
Because this whole tutorail environment is in nested lxc containers, I can safely use  ammended `/etc/hosts/` file which maps
the target containers' ip addresses to names, which then match a simple `~/.ssh/config ` file

I use a zsh alias to produce a block of name-mappins suitable for the `hosts` file
```
% alias lxc_hosts="lxc ls -c n4 --format csv | cut -d' ' -f1 | awk -F',' '{print \$2, \$1 \"-ansible\"}'"
% lxc_hosts
10.69.189.150 centos-ansible
10.69.189.56 one-ansible
10.69.189.216 three-ansible
10.69.189.215 two-ansible
10.69.189.192 ws-ansible
```

```
% cat ~/.ssh/config
Host centos*
	User lnxcfg

Host *ansible
	IdentityFile ~/.ssh/ansible

Host github.com
	User pjgoodall
	####

Host *
	User ubuntu
	IdentityFile ~/.ssh/peter_tutorial
	StrictHostKeyChecking=no
```

### Create the initial inventory file

```
lxc ls -c 4 --format csv| cut -d" " -f1 > inventory
```

or

```
% cat ~/.ssh/config| grep '\-ansible'| cut -d" " -f2 > inventory
```
 ### ansible `-- become option`

no password on the lxc servers, so password is empty when asked, but not really required.

### CentOS 8 container configuration

Had to do some extra work to get sftp to work - see comments in [setup-lnxcfg-user.sh](./setup-lnxcfg-user.sh)

Out of the box, the centos/8 container is not very cooperative:
- No `su`  so ansible `become:` fails
- No ssh-serrver
- Only a root user

[Learning Ansible with CentOS 7 Linux](https://brad-simonin.medium.com/learning-ansible-with-centos-7-linux-12461043fd02) by Bradley Simonin provides guidance. I modified his script here [setup-lnxcfg-user.sh](./setup-lnxcfg-user.sh), to install the missing software as well as implement Bradley's process. I've left my example public key in-place. 

To set up a centos container for ansible:

```
# Create the container
lxc launch images:centos/8 --profile default centos

# Push the initialization script to the container
lxc file push ./setup-lnxcfg-user.sh centos/root/

# Execute the set-up script
lxc exec centos -- /bin/bash ./setup-lnxcfg-user.sh
```

The you need to update the relevant sections of your ansible server's `~/.ssh/config`. Should look a bit like this:

```
Host centos-ansible
    HostName 10.69.189.138
    User lnxcfg

Host *ansible
	IdentityFile ~/.ssh/ansible

```



