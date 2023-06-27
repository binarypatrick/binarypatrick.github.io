---
layout: post
title: "Bitwarden Automated Backup"
date: 2023-06-26 12:00:00 -0400
category: "Service Setup"
tags: ['linux', 'bitwarden', 'backup']
---

## Purpose

## Prerequisites

You will need `unzip` and `cron` simply to install the Bitwarden CLI.

```bash
sudo apt install unzip cron -y
```

## Installing the Bitwarden CLI

Unfortunately it does not exist in the distro repo or flatpak. Though it is in snap, if you are interested in such things. I installed it manually through a handy script I wrote. It will make updating it manually later a bit easier it at least. Either way the CLI binary will need to be somewhere we can run it.

```bash
curl -L -o bw.zip "https://vault.bitwarden.com/download/?platform=linux&app=cli"
unzip bw.zip
sudo mv ./bw /usr/local/bin
rm bw.zip
```

## Bitwarden User

Before we begin we will need to create a new user for running the script. This user will securely store the environment variables as well. Unfortunately even though we have the API key, Bitwarden still requires your vault password. I assume this is for actually decrypting the vault. To store them securely, I put them in the service user's `~/.bash_profile`. This way only that user, and sudoers/root will have access and the job won't need to run as root. Obviously plaintext isn't ideal, but for the script to use it, it must be read somewhere. No amount of encryption would change that. Also note this machine should be isolated on your network from other services. 

To create your user run the following:

```bash
sudo adduser \
   --system \
   --shell /bin/bash \
   --group \
   --disabled-password \
   --home /home/bitwarden \
   bitwarden
```

## Bitwarden API Credentials

To get your API key log into your Bitwarden web vault. From the user menu in the upper right, go to _Account Settings_. Then on the left hand menu go to _Security_, then _Keys_ in the top menu. This should bring you to _Encryption Key Settings_ and at the bottom of the page _API Key_. Your account will only have one set of client ID/secret. Click _View API Key_ to retrieve it. You'll need these values to add to the `.bash_profile` along with your vault password.

```bash
sudo touch /home/bitwarden/.bash_profile
sudo chmod 600 /home/bitwarden/.bash_profile
sudo nano /home/bitwarden/.bash_profile
```

```conf
export BW_CLIENTID="<your_client_id>"
export BW_CLIENTSECRET="<your_client_secret>"
export BW_PASSWORD="<your_vault_password"
export BW_NOTIFICATION_EMAIL="<your_notification_email_address>"
```

## Setting up the Script

The script needs to be put somewhere the Bitwarden user can read it, and it needs to be set as executable.

```bash
REPO_BACKUP_SCRIPT=https://raw.githubusercontent.com/BinaryPatrick/BitwardenBackup/main/backup.sh
sudo curl -L -o /home/bitwarden/backup.sh $REPO_BACKUP_SCRIPT
sudo chown bitwarden /home/bitwarden/backup.sh
sudo chgrp bitwarden /home/bitwarden/backup.sh
sudo chmod 744 /home/bitwarden/backup.sh
```

The script also includes an email notification if the vault fails to unlock or authentication fails. You'll need to [set up postfix to send email](/posts/configuring-postfix-with-gmail/).

## Adding to `crontab`

Now that the script is in place, we can add it to `crontab`. We need to do a little hand holding to make sure the `crontab` environment can see the bw binary, and will use the environment variables we configured. Also notice, our script requires an output directory to run. We'll set this to `/home/bitwarden`. Feel free to configure this to use a mounted share or something though. Remember the output is encrypted using the vault password so it is relatively safe to have on a restricted file share.

```bash
sudo su bitwarden
crontab -e
```

```conf
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
0 0 * * * BASH_ENV=/home/bitwarden/.bash_profile /bin/bash /home/bitwarden/backup.sh /home/bitwarden
```

## Validate Decryption

An untested backup is no backup at all. Make sure to try and decrypt the file that is created using the decryption script once you have a backup. It should create a json file.

```bash
REPO_DECRYPTION_SCRIPT=https://raw.githubusercontent.com/BinaryPatrick/BitwardenBackup/main/decrypt.sh
sudo curl -L -o decrypt.sh $REPO_DECRYPTION_SCRIPT
sudo chmod +x decrypt.sh
```

When you run the script, pass the filename of the file you want to decrypt and you will be prompted for your vault password.

```bash
./decrypt.sh bw_export_xxxxxxxxxxxxxxx.enc
```

## Resources
- https://www.digitalocean.com/community/tutorials/send-email-linux-command-line
- https://easyengine.io/tutorials/linux/ubuntu-postfix-gmail-smtp/
- https://bitwarden.com/blog/how-to-back-up-and-encrypt-your-bitwarden-vault-from-the-command-line/
- https://bitwarden.com/help/cli-auth-challenges/