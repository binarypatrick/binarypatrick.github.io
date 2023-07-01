---
layout: post
title: "Configuring Postfix with Gmail"
date: 2023-06-26 00:00:00 -0400
category: "Service Setup"
tags: ["linux", "email", "gmail"]
---

Email is often a critical component notification for jobs and other things in linux. This is how I set up [Postfix](https://www.postfix.org/) on most of my instances. This guide is geared towards Debian based distros but can be translated for others.

## Required Software

```bash
sudo apt install postfix mailutils -y
```

Both Postfix and mailutils make this much easier

## Configuration

Open the Postfix configuration file:

```bash
sudo nano /etc/postfix/main.cf
```

Append the following to the end:

```conf
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/gmail_credentials
smtp_sasl_security_options = noanonymous
smtp_use_tls = yes
```

Notice that instead of adding your gmail credentials directly, we're adding them to a file named `gmail_credentials`. Your credentials will depend on whether you use multi-factor authentication (MFA/2FA) or not. If you are now, then you can just put your credentials directly into this file, otherwise you'll need to create an app password. It is highly recommended to use multi-factor authentication whenever possible so I'll assume you are.

> It is highly recommended to use multi-factor authentication whenever possible
{: .prompt-tip }

Go to your [Google account page](https://myaccount.google.com/) and select _Security_ from the side navigation. Then in the center, select _2-Step Verification_. After verifying it's really you, scroll to the bottom and find _App Passwords_. From this page you can generate a new app specific password. I often select custom from the drop down and give it a descriptive name, then select _Generate_.

Remember to copy the generated password, because you will never see it again. Now you can use this new generated password instead of your actual password in your `gmail_credentials` file.

```bash
sudo nano /etc/postfix/gmail_credentials
```

```conf
[smtp.gmail.com]:587 <username>@gmail.com:<password>
```

Once you have your credentials file created, you need to secure it add enroll it with postfix.

```bash
sudo chmod 400 /etc/postfix/gmail_credentials
sudo postmap /etc/postfix/gmail_credentials
```

Now you can reload the Postfix service

```bash
sudo systemctl reload postfix
```

## Testing

Once reloaded, you should be able to send email from the CLI

```bash
echo "This is a test email" | mail -s "Test of Postfix using Gmail" <your@email.address>
```
