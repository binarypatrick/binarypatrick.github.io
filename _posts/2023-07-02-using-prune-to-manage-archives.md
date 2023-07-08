---
layout: post
title: "Using Prune to Manage Archives"
date: 2023-07-02 00:00:00 -0400
category: "Service Setup"
tags: ["linux", "backup", "FOSS"]
---

Often times when using a tool like rsync to make backups you end up with lots of archives, and no way to keep them managed over time without a manual process. Prune is a simple tool that lets you remove prune archives in a folder, deleting any archives not matching the specified retention options. Any file type can be an archive and prune allows you to specify which files are in scope to be pruned.

[https://github.com/BinaryPatrick/Prune](https://github.com/BinaryPatrick/Prune)

## Install

To install Prune you can go to the [releases page](https://github.com/BinaryPatrick/Prune/releases) and download the latest release for your environment. You can also just clone the repo and compile it yourself with the steps in the readme. If you are running Linux on x64 you can also run the following commands to download and install Prune.

> Don't just take my word for it. Always inspect the code that will be running on your machines, especially from an untrusted and unsigned source.
{: .prompt-warning }

```bash
curl https://raw.githubusercontent.com/BinaryPatrick/Prune/main/scripts/install-linux-x64.sh | sudo bash
```

## Usage

Once it's installed, you should be able to run prune simply by typing in `prune`, **but don't**. Prune is designed to delete things, and you should always run it with `--dry-run` and `--verbose` until you're certain what will be pruned.

![prune help](/assets/img/using-prune-to-manage-archives/prune-help.png)

To use prune, you will need to provide a pruning path and some retention value. Keep in mind prune is not recursive, so it will only purge files directly in the path you give. Prune also uses last modified date, not created date to evaluate what files to keep and prune. This allows for incremental backups with updates to be properly evaluated. You must specify some level of retention for prune to run.

```bash
prune --path /home/patrick/test/ --keep-last 10 --dry-run --verbose
```
Example output:

![prune example output](/assets/img/using-prune-to-manage-archives/prune-example-output.png)

You can also provide more file filtering using `--prefix` and `--ext`.

```bash
prune --path /home/patrick/test/ --keep-last 10 --prefix backup_ --ext tar.gz --dry-run --verbose
```

Finally you can scope in the exact retention you want using the different retention interval filters.

```bash
prune --path /home/patrick/test/ --keep-daily 7 --keep-weekly 2 --dry-run --verbose
```

So far I have tested this with Debian 12 and Raspbian and it runs great.

Happy pruning!
