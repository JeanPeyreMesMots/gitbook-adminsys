---
description: My first steps on Active Directory :)
---

# 💻 Introduction and setup

When I started, I knew nothing about Active Directory. I knew it was the directory and authentication service (LDAP/Kerberos) used to manage Windows machines in companies, but I had never actually worked with it.&#x20;

So I reached out on Discord to a sysadmin working in the public sector, who kindly designed a lab for me covering the basics of setting up an Active Directory server.

The goal was to build a server for a company called "MesMots" (a nod to my username :D), with user accounts organized into groups. Each group would get specific permissions on folders hosted on an SMB share. Here was the scenario:

"_**MesMots is a fictional company specializing in teaching literature and writing. It has 24 employees spread across four distinct departments. This lab builds on that structure to practice identity and access management on Windows Server 2019.**_"

The lab objectives were:

* Create and organize the Active Directory (OUs, Groups, Users)
* Configure an SMB share with a network drive mapped on X:
* Apply granular NTFS permissions per department
* Understand segregation of access rights in a company
* Implement Windows Server security best practices

The environment runs on **Windows Server 2019**, chosen because it is still widely deployed in companies at the time of writing. The server is configured as a domain controller, handling users, groups, organizational units and file shares.

The work had to follow this organization chart:

<figure><img src="../../../.gitbook/assets/image (71).png" alt=""><figcaption></figcaption></figure>

Without further ado, let's get started!