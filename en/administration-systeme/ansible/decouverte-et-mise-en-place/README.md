---
description: Getting started with Ansible
---

# ☁️ Introduction and setup

As part of my career transition into system administration, I'm publishing here the notes I took while following **Cocadmin's Ansible course** (in French), a french-speaking canadian sysadmin with pure gold content. All credit for the course content, the lab scenario and the example application goes to him. These pages are my personal notes and the issues I ran into along the way.

The course was extremely useful to get back up to speed with Ansible, which I had already used during my 2022 work-study program to deploy and provision a VM that copied files from an old production server to a new one.

The first step is to set up a test environment to learn and practice Ansible.

Next, the objectives in order are:

* Deploy several test virtual machines on my machine with Multipass.
* Keep name resolution working despite IP changes caused by DHCP.
* Prepare the servers for Ansible's SSH connections.
* Install Ansible and check that the inventory works.

For reference, my setup is: Multipass VMs (KVM), inside an Ubuntu VM, itself running on VirtualBox, on Windows 11.