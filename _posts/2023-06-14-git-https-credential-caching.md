---
layout: post
title: "Git Credential Caching"
date: 2023-06-14 00:00:00 -0500
category: "General"
tags: ['git']
---

Having a local git repo configured, I haven't yet set up ssh authentication. Instead I use HTTPS with a username and password. It can be tiresome typing in the login credentials over and over, so I typically cache the credentials with a reasonable timeout.

```bash
git config --global user.name "patrick"
git config --global credential.helper 'cache --timeout=3600'
```

I also make sure my .git/config contains the username so I don't need to type that in over and over.

```conf
https://<USERNAME>@github.com/path/to/repo.git
```

More helpful information can be [found here](https://www.shellhacks.com/git-config-username-password-store-credentials/).