---
layout: post
slug: autostart-api-spa
title: "Start a SPA and it's API in one click"
pubDatetime: 2021-06-16T12:00:00
tags:
  - Powershell
  - Terminal
  - Angular 2
  - .NET
description: 'I''ve been using start or jump scripts for a long time to start up my API and SPA at the same time. With project TYE on the horizon, I thought I would share more about my more simplistic, "poor mans", implementation.'
---

I've been using start or jump scripts for a long time to start up my API and SPA at the same time. With [project TYE](https://devblogs.microsoft.com/aspnet/introducing-project-tye/) on the horizon, I thought I would share more about my more simplistic, "poor man", implementation.

<!--more-->

## Notes

It is important to note, a lot of configuration information is specified ahead of time in different files to make this work. For ASP.Net, the ports are configured in the _launch.json_ and for node, proxy information is set up in a _proxy.config.json_ file. Below is a simple example of the proxy config I use most often to route api requests to the ASP.Net port. It is also good to remember both apps are running locally under localhost.

```json
// proxy config
{
  "/api": {
    "target": "http://localhost:5000",
    "secure": false,
    "logLevel": "debug",
    "changeOrigin": false
  }
}
```

it's also important to declare the script you want node to run in scripts section of your package.json. Typically I use:

```json
"start": "ng serve --proxy-config proxy.config.json"
```

## Humble beginnings

For my first attempt at making my start script, I created a batch file that started two scripts. One new Powershell window for `dotnet watch run` and another for `npm run start`. This had some hiccups initially. The windows would close if the Powershell stopped running, which was problematic for collecting errors. Also the command prompt window would linger. After a few iterations, I was able to solve those problems and landed on this:

```batch
cd ./src/api
start powershell.exe -NoExit dotnet watch run
cd ../spa
start powershell.exe -NoExit npm run start
```

![Original script result](../../../assets/images/2021/autostart-api-spa-app/original-script.png)

## Next evolution

This script worked well from ASP.Net Core 1.3 and Angular 2, all the way through today with .Net 5 and Angular 12. Recently though, I started to dabble in Powershell Core and Windows Terminal which led me to a more efficient approach to starting and managing my scripts.

```batch
set apiLocation=.\src\api
set spaLocation=.\src\spa

wt --title "dotnet watch run" -d %apiLocation% powershell -noExit "dotnet watch run"; --title "npm run start" -d %spaLocation% powershell -noExit "npm run start"
```

With this one line, I can start a Windows Terminal with both scripts running in tabs. This allows me to keep both together and but also manage the window as one unit. The commands are similar, but the tabs are labeled, so I can quickly jump where I need to look. Also, this is far more expandable if I needed to run other startup scripts together in the future.

![New script result](../../../assets/images/2021/autostart-api-spa-app/new-script.png)
