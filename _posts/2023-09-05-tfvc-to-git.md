---
layout: post
title:  "Migrate TFVC to Git in Azure DevOps"
date:   2023-10-03 19:23:05 +0100
---

### In this post

- [Overview](#overview)
- [TFVC Import Tool](#first-option-tfvc-import-tool)
- [Limitation of using TFVC Import Tool](#limitation-of-using-tfvc-import-tool)
- [GIT-TFS Tool](#second-option-git-tfs-tool)
- [Summary](#summary)

## Overview

Migrating from TFVC to Git in Azure DevOps can bring numerous advantages, including better collaboration, easier branching and merging, and improved performance. With Git's distributed nature, teams can work more independently and efficiently, while still having full visibility into changes and history.

If you're considering migrating from TFVC to Git in Azure DevOps, this post will guide you through the process step-by-step by introducing two main options for migration.

## First Option: TFVC Import Tool

It is an out of box tool in Azure DevOps, The migration is very simple and great for small simple TFVC repositories.

Check out the image below for the migration steps:

![import-TFVC.gif](/assets/images/tfvc-git/tfvc-import-tool.gif)

## Limitation of using TFVC Import Tool:

Using this option has some limitations, including:

1. The imported repository and associated history cannot exceed `1GB` in size.
1. You can import up to only `180` days of history.

What if your repo size exceeds 1 GB, or you need to import all history which may exceed 180 days? 
You may want to consider the second option: using the GIT-TFS tool.

## Second Option: GIT-TFS Tool
It is a two-way bridge between TFVC and Git, and you can use it to perform a migration. It is suitable for a migration with a full history, more than the 180 days that the Import tool supports.

**Here are the steps to migrate from TFVC to Git using GIT-TFS:**

1. Installation: If Chocolatey (Package Manager for Windows) is already installed on your machine, run `choco install gittfs`. 

    If Chocolatey is not installed, you can install it from [here](https://docs.chocolatey.org/en-us/choco/setup).

2. To confirm that git-tfs is installed on your machine, run `git tfs --version`.

3. Run git tfs clone, and pass the TFVC repository’s URL and repository path as arguments.

    The URL should be as following:

    `git tfs clone https://dev.azure.com/{Organization} $/{Repo}`

    For example, if you want to migrate a repository named `example` from `https://dev.azure.com/TestOrg` run 

    `git tfs clone https://dev.azure.com/TestOrg $/example`

After running this command, you will have a local Git repo migrated from TFVC. You can then push the resulting local Git repository to any Git platform of your choice, such as Azure DevOps, GitHub, Bitbucket, or others.

![import-TFVC-tfs.gif](/assets/images/tfvc-git/git-tfs.gif)

**❗ Note ❗:** The cloning process can take a significant amount of time, depending on the size of your TFVC repository and its history.

## Summary

In summary, with the instructions provided in this post, you can successfully migrate from TFVC to Git in Azure DevOps. By doing so, you can take advantage of the benefits that Git provides and improve your team's productivity and collaboration.
