---
layout: single
title: "Creating a Local YUM Repository on Rocky Linux"
subtitle: ""
date: 2023-11-10 18:15:00 +0100
background: '/image/01.jpg'
tags: ['linux']
---

{% raw %}

# Creating a Local YUM Repository on Rocky Linux

Local YUM repositories are crucial for systems where internet access is limited, or package deployment needs tight control. This short article guides you through setting up a local repository on Rocky Linux.

## Unpacking RPM File

First, unpack the desired RPM files into a specific folder. This step assumes you have already downloaded or have access to the RPM files you wish to include in your local repository.

## Installing Necessary Tools

To create and manage your local repository, you will need yum-utils and createrepo. Install these using the following commands:

````bash
yum install yum-utils createrepo
````

These tools provide utilities to manage your YUM configuration and create the metadata needed for a YUM repository.

## Creating Your Local Repository

Next, use createrepo to create a local repository. This command turns a directory of RPMs into a repository that YUM can use.


````bash
createrepo /your/local/folder/with/rpms
````

Replace ``/your/local/folder/with/rpms`` with the path to the directory where you have stored your RPM files.

## Managing YUM Cache

Before using your new repository, it's necessary to clear YUM's cache and create a new one. This ensures that YUM recognizes your newly created repository.

````bash
yum clean all
yum makecache
````

## Configuring YUM to Use the Local Repository

To use your new local repository, disable all other repositories and enable your newly created one. Here’s how to do it:

````bash
yum --disablerepo="*" --enablerepo="myrepo"
````

Replace myrepo with the real name of your repository. Note that this command should be followed by an action like an installation command; otherwise, it just sets up the configuration without performing any tasks.

If you do not know the name of the local repo, use ``yum repolist`` command to get all available repos.

## Conclusion

Setting up a local YUM repository on Rocky Linux is straightforward and extremely useful for controlled environments or offline systems. By unpacking RPMs, installing yum-utils and createrepo, initializing your local repository, managing YUM cache, and configuring YUM to use the local repository, you ensure a reliable and controlled package management setup. Remember to regularly update your local repository with new or updated RPMs as needed.

{% endraw %}