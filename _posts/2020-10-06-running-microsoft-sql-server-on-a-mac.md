---
layout: post
title:  Running Microsoft SQL Server... on a Mac!
description: A few years ago I switched my main work machine from Windows to Mac – despite the reliance I have on SQL...
date:   2020-10-06 15:01:35 +0300
image:  '/images/posts/2020-10-06-running-microsoft-sql-server-on-a-mac/apples.jpeg'
tags:   [docker, sql]
---

## 👋 Introduction
A few years ago I switched my main work machine from Windows to Mac – despite the reliance I have on SQL Server to run 
our application (Well, there’s also the dev-only H2 database, but even then I knew that would not fly due to some ‘subtle’ differences).

![SQL Error]({{site.baseurl}}/images/posts/2020-10-06-running-microsoft-sql-server-on-a-mac/sql-error.png)

At the time I had no idea what containerisation was – so I thought my only option was to run my database in a Windows 
VM or use Wine to run SQL Server natively. Thankfully I was then introduced to docker – it allowed me to run my database 
on the macOS side (well, technically within a Linux container), rather than faffing around with a Windows VM, now I can 
gladly say I only need a Windows VM for one task, and that’s because I am yet to find a replacement a powerful as SSMS 
for comparing query plans.

## 🐳 Installing Docker
Installation of docker is pretty straightforward, just follow 
[the instructions on the docker site](https://docs.docker.com/docker-for-mac/install/). Once complete, the Docker 
'whale' icon should be visible in the macOS menu bar.

![Menu Bar]({{site.baseurl}}/images/posts/2020-10-06-running-microsoft-sql-server-on-a-mac/mac-menu-bar.png)

## 🏃‍♂️ Pull & Run the SQL Server Docker image
In a terminal window of your choice (I recommend [iTerm2](https://iterm2.com/)) – run the following;

#### Pull the container image from Docker Hub
```bash
sudo docker pull mcr.microsoft.com/mssql/server:2019-latest
```

#### Run the image
```bash
sudo docker run \
   -e "ACCEPT_EULA=Y" \
   -e "SA_PASSWORD=<YourStrong@Passw0rd>" \
   -p 1433:1433 \
   --name sql1 \
   -d mcr.microsoft.com/mssql/server:2019-latest
```

| Parameter                                     | Description                                                                                                                                                    |
|-----------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `-e “ACCEPT_EULA=Y”`                          | Set ACCEPT_EULA to confirm you accept the [end user licensing agreement](https://go.microsoft.com/fwlink/?LinkId=746388), this is required to start the image. |
| `-e “SA_PASSWORD=<YourStrong@Passw0rd>”`      | Specify your own strong password, again this is required to start the image.                                                                                   |
| `-p 1433:1433`                                | Map a port number on the host environment (your machine) to with a TCP port on the container (second number)                                                   |
| `-name sql1`                                  | 	A name for the container. If not specified, a [random one](https://github.com/moby/moby/blob/master/pkg/namesgenerator/names-generator.go) will be generated. |
| `-volume /Users/mark/DockerShare:/HostShare/` | Map a shared folder from the Host OS to the docker container (Very useful for transferring database backups!)                                                  |

#### Connect to Docker SQL Server with a SQL Editor
So, you may have guessed by now – but not only will SQL Server not work on macOS, but neither will [SQL Server Management Studio](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15)! But fear not, Microsoft still has our backs – I have been using [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15) to connect to and manage my SQL Server container, and have found it can do (almost) everything I need. To connect, simply input the login credentials specified when running the container.

![Azure Connection]({{site.baseurl}}/images/posts/2020-10-06-running-microsoft-sql-server-on-a-mac/azure-connection.png)

## Or, the lazy way
I have written a quick Python script to install my DB, alongside some useful Stored Procedures I use for monitoring (you should too - but that's a story for another day!) - and this can be found on my [GitHub](https://github.com/MTJB/scripts/blob/main/database/setupDb.py)

## 💅 Summary
As you can see – it’s very simple to get an instance of SQL Server running in Docker. Within a few short minutes, you can have the Server up and running – and likewise – replace an existing SQL Server image (if you’re anything like me, replacing the image will be pretty common if you bloat it with unused database backups!).