---
layout: post
title: "Using curl with SNI"
date: 2024-05-01 00:00:00 -0400
category: "General"
tags: ["linux", "bash"]
---

This is a short little reminder for myself, when using curl to make requests to local things and spoofing SNI, use the following command.

```bash
curl -vik --resolve google.com:443:127.0.0.1 https://google.com
```

Happy Curling