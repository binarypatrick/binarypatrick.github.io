---
layout: post
title: "Bitwarden Automated Backup"
date: 2023-06-26 12:00:00 -0400
category: "Service Setup"
tags: ['linux', 'bitwarden', 'backup']
---

## Purpose

## Installing the Bitwarden CLI
```bash
curl -L -o bw.zip "https://vault.bitwarden.com/download/?platform=linux&app=cli"
unzip bw.zip
sudo mv ./bw /usr/local/bin
rm bw.zip
```

## Bitwarden API Credentials

/root/.bash_profile

## Running the Script

```text
# Before you run this script you will need to have the following environment variables set
# BW_CLIENTID           // Bitwarden API app client ID
# BW_CLIENTSECRET       // Bitwarden API app client secret
# BW_PASSWORD           // Bitwarden login password
# BW_NOTIFICATION_EMAIL // Email address used for notification if job fails

bw login --apikey

export BW_SESSION=$(bw unlock --raw $BW_PASSWORD)

if [ "$BW_SESSION" == "" ]; then
        echo "The automated Bitwarden backup failed when trying to unlock the vault" | mail -s "Bitwarden Backup Failed" $BW_NOTIFICATION
        bw logout
        exit 1
fi;

EXPORT_OUTPUT_BASE="bw_export_"
TIMESTAMP=$(date "+%Y%m%d%H%M%S")
ENC_OUTPUT_FILE=$EXPORT_OUTPUT_BASE$TIMESTAMP.enc

bw --raw --session $BW_SESSION export --format json | openssl enc -aes-256-cbc -pbkdf2 -iter 1000000 -k $BW_PASSWORD -out $ENC_OUTPUT_FILE

bw logout
unset BW_SESSION
```

## Adding to `crontab`

```bash
sudo crontab -e
```

```conf
PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
0 0 * * * BASH_ENV=/root/.bash_profile /bin/bash /root/backup.sh
```

## Validate Decryption

```bash
OUTNAME=$(basename $1 .enc).json
openssl enc -aes-256-cbc -pbkdf2 -iter 1000000 -d -nopad -in $1 -out $OUTNAME
```

## Resources
- https://www.digitalocean.com/community/tutorials/send-email-linux-command-line
- https://easyengine.io/tutorials/linux/ubuntu-postfix-gmail-smtp/
- https://bitwarden.com/blog/how-to-back-up-and-encrypt-your-bitwarden-vault-from-the-command-line/
- https://bitwarden.com/help/cli-auth-challenges/