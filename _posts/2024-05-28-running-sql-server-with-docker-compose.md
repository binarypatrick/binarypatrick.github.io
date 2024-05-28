---
layout: post
title: "Running SQL Server using Docker Compose"
date: 2024-05-28 00:00:00 -0400
category: "General"
tags: ["linux", "docker", "sqlserver", "sql"]
---

This was much easier than I thought. Using the following docker compose file, you can run SQL Server locally in docker. It is critical to use a volume for the `mssql` directory, <u>**otherwise you will lose all your data when the container restarts**</u>. The image is first party (published by Microsoft) [listed in docker hub](https://hub.docker.com/_/microsoft-mssql-server/), though the container is pulled down from Microsoft's registry directly I think.

```yaml
# docker-compose.yaml

services:
  sql-server-db:
    container_name: sql-server-db
    image: mcr.microsoft.com/mssql/server:2022-latest
    ports:
      - "1433:1433"
    environment:
      SA_PASSWORD: ${SA_PASSWORD}
      ACCEPT_EULA: "Y"
      UID: 1000
    volumes:
      - ./mssql:/var/opt/mssql:rw
```

You'll also need to change the _sa_ account password, or create an `.env` file.

```conf
# .env

SA_PASSWORD=some_random_password
```

Once this is up and running, you can access it normally through [SSMS](https://learn.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) or [Azure Data Studio](https://learn.microsoft.com/en-us/azure-data-studio/download-azure-data-studio). The connection will require you to use encryption, though that should be handled automatically.