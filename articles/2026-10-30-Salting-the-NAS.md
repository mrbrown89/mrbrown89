# Salting the NAS

> Originally published in *2600: The Hacker Quarterly*.

By Matt Brown

## Building a Home Rolled NAS

I was asked to build a home rolled NAS system at work that would act as a disaster recovery system in the event of the main NAS failing and also handle backup tasks. I rubbed my hands together with glee because I absolutely love storage systems. I couldn’t tell you why. Just something about them. I used to work with IBM’s GPFS filesystem which I would have used, however, it’s extremely expensive. I also love open source and, to me at least, the rebellious side of it.

So with my server and JBOD I built a ZFS based NAS system running Samba using WinBind for domain authentication and NFS both of which use CTDB to handle network interfaces.

## Storage Architecture Development

After purchasing a server and a JBOD, I loaded the server with some NVMe disks to serve as a metadata vdev, which is a group of disks combined together in a software RAID configuration managed by ZFS. I chose to have a dedicated vdev for metadata because the majority of data being stored is animation frames. When navigating directories with 20,000 or more frames, file explorers can struggle before showing all the files. Having metadata on fast disks ensures that these frames appear instantly.

I loaded my JBOD with 18TB HDDs and striped them into several vdevs, each configured as RAID-Z2. RAID-Z2 is similar to RAID 6 in that it can tolerate two disk failures, but a third failure would result in data loss. Keeping the vdevs relatively small helps reduce resilvering time, which is the process ZFS uses to rebuild a vdev when a disk is replaced.

## Creating the ZFS Pool and Filesystem

My metadata vdev and HDD vdevs together form the ZFS pool where the filesystem is mounted. From there, I created datasets to manage the data, set up Samba and NFS for file sharing with CTDB handling the tow network interfaces, and configured WinBind for Active Directory authentication. The NAS also runs the company’s chosen backup product, P5, which handles daily backups to an S3 bucket and LTO tape.

## From Storage Server to Multi Role Machine

Over time, the NAS started taking on additional roles. A few containers and VMs running in KVM were set up on the machine, increasing the number of moving parts and the number of configuration files. Because the NAS handles critical company backups, I can’t just poke around the system without risking disruption to an essential piece of infrastructure.

## The Problem with Manual Configuration

So how do I make changes to the system safely and how do wrangle all of the configuration, software updates etc in a clean way that can be reproduced? Handling systems by manually editing file after file becomes tedious very quickly. Editing files manually is also error prone, dangerous in production and pain to replicate from test environment to production. Breaking samba or breaking CTDB can lead to a very boring time!

## The Case for Automation and Testing

Yup, test environments. I’m a big believer in running test builds of a new configuration in virtual machines and never testing on a production system. I also like using these VMs to learn in a safe environment. But its a real pain to spin up a new VM and run all the commands manually. Its also a pain to test configurations and scenarios in your dev environment and then manually do the same thing on a production system.

Wouldn’t it be great if I could use some sort of automation system where I can declare configurations in files that I can test and then copy to my production system? Wouldn’t it be even better if I could have all of these configs in a repo when I can have a dev branch for building configs, a test branch which can be tested on VMs and even automated with Jenkins and then a production branch where merged config files have been tested to work and so can be applied safely to the production system. Sounds too good to be true!

## Enter SaltStack

At a previous employer we made SAN systems using IBM’s GPFS as the filesystem. During my time here I learned a lot about filesystems but also how to manage Linux systems at scale using automation with tools like SaltStack or simply Salt.

## How SaltStack Works

Salt is an open source Infrastructure as Code (IaC), configuration management, and automation tool. It allows you to control and manage multiple servers (called minions) from a single central server (the master).

With Salt, you define declarative state files that describe exactly how a system should be configured. When these states are applied, Salt ensures the target system matches that description — installing packages, managing files, starting services, and so on.

So how does Salt actually work? Its configuration system rests on three main parts:

### Grains

Information about each minion, such as its operating system, CPU, memory, IP addresses, and roles. Grains are gathered automatically by the minion, though you can define custom ones if needed.

### Pillars

Structured data defined on the master and passed securely to minions. Pillars are often used for sensitive values like passwords or keys, but can hold any form of configuration data.

### States

YAML files that describe the desired configuration of your systems, for example which packages should be present or which services should be running.

Taking all of the knowledge I had gained from years of working with storage systems made by others, I changed my NAS to use salt to handle all of the configuration files with the salt configs all hosted in GitHub.

## Automation: Jenkins and Git

Hosted on a Proxmox server I have a mini version of this NAS running alongside a container running Jenkins.

When I need to make a change I can create a feature branch in my local repo where I can merge into a testing branch once I’m happy with my changes. I can then pull my testing branch to my salt development server and then use Jenkins to run salt calls and perform checks against my virtual NAS. Once all my tests have run and I’m happy with the changes, the changes can be merged into my main branch, a pull request run on my production NAS and then run a salt call to apply the changes safe in the knowledge the salt states have been tested.

## Giving Back to Open Source

I mentioned earlier that I love open source projects and I try to use open source projects as far as possible and I’ve always wanted to give back to the open source community so I’ve open sourced a NAS build using salt! You can find the project here:

<https://github.com/mrbrown89/son-of-anton>

## Why “Son of Anton”?

What’s with the name? I’m a huge fan of the TV show Silicon Valley. I named my NAS at work Anton and since in the show the computer Anton dies and a new one is created called Son of Anton I thought why not call this project Son of Anton as its born out of the work and running configuration on Anton!

## Play, Learn, and Scale

Using the code, you can run a single script (providing you’ve set the VM up correctly) on a Debian based VM to create a working ZFS filesystem complete with NFS exports and SMB shares that you can play around with.

The tools and methodology used in this project can be used at scale to manage a NAS or anything else that needs management. I also use salt to manage software deployment in the animation studio that I work in. Salt allows me to deploy lots of software across all my machines in parallel taking upgrade work from weeks to a matter of hours.
