---
order: 2
title: "Local Development Setting"
---

# Local Development Setting

> Environment Requirement

- golang 1.20 +
- **nodejs 18.8.0 +**
- **mysql 8.0 +** | MariaDB 10.7 + | Postgres 14 + (**Postgres 15 + recommended**)
- redis 6.0 +
- [go-swagger](https://goswagger.io/install.html)
- [Simple Admin Tool](/guide/basic-config/simple-admin-tools.md)

::: info
It is recommended to develop under linux, because the make command is required. We develop in Ubuntu 22.10. \
**`Windows` users are recommended to develop in the [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) environment, you can also cofigure the environment via [Windows](/guide/FAQ.html#how-to-configure-the-windows-environment).**
:::

## Backend Setting

### simple admin core

simple admin core is the core service for simple admin. It offers user management, authorization,
menu management and API management and so on. It must be running.

::: info Default Account
username: **admin**  
password: **simple-admin**
:::

> Clone the core code

```bash
git clone https://github.com/suyuan32/simple-admin-core.git
```

### Local development setting

#### API Service

##### Notice: You should add core_dev.yaml for development to avoid conflicting with core.yaml in production.

> Add api/etc/core_dev.yaml

```yaml
Name: core.api
Host: 0.0.0.0 # ip can be 0.0.0.0 or 127.0.0.1, it should be 0.0.0.0 if you want to access from another host
Port: 9100
Timeout: 30000

Auth:
  AccessSecret: jS6VKDtsJf3z1n2VKDtsJf3z1n2 # JWT encrypt code
  AccessExpire: 259200 # seconds, expire duration

Log:
  ServiceName: coreApiLogger
  Mode: file
  Path: /home/ryan/logs/core/api # log saving path，use filebeat to sync
  Level: info
  Compress: false
  KeepDays: 7 # Save time (Day)
  StackCoolDownMillis: 100

RedisConf:
  Host: 127.0.0.1:6379 # Change to your own redis address
  Type: node
  # Pass: xxx  # You can also set the password

# The main difference between k8s and local development
# Core service
CoreRpc:
  Endpoints:
    - 127.0.0.1:9101 # The same as Core RPC's address.
  Enabled: true # Whether to enable the service

# Scheduled Task RPC service
JobRpc:
  Endpoints:
    - 127.0.0.1:9105 # The same as Job RPC's address.
  Enabled: false # Whether to enable the service

Captcha:
  KeyLong: 5 # captcha key length
  ImgWidth: 240 # captcha image width
  ImgHeight: 80 # captcha image height

DatabaseConf:
  Type: mysql # support mysql, postgres, sqlite3
  Host: "127.0.0.1" # change to your own mysql address
  Port: 3306
  DBName: simple_admin # database name, you can set your own name
  Username: root # username
  Password: "123456" # password
  MaxOpenConn: 100 # the maximum number of opened connections in the  connection pool
  SSLMode: disable # in postgresql, disable or require
# DBPath: /home/data/sqlite.db # Database Path, you must set it when you use sqlite3.

# casbin rule
CasbinConf:
  ModelText: |
    [request_definition]
    r = sub, obj, act
    [policy_definition]
    p = sub, obj, act
    [role_definition]
    g = _, _
    [policy_effect]
    e = some(where (p.eft == allow))
    [matchers]
    m = r.sub == p.sub && keyMatch2(r.obj,p.obj) && r.act == p.act
```

::: warning

> Small website use endpoint connect directly

```yaml
CoreRpc:
  Endpoints:
    - 127.0.0.1:9101 # the same as rpc address
```

> it does not need service discovery，when you develop locally, you should also use this mode. There can be several endpoints.
> :::

> Add rpc/etc/core_dev.yaml

```yaml
Name: core.rpc
ListenOn: 0.0.0.0:9101 # ip can be 0.0.0.0 or 127.0.0.1, it should be 0.0.0.0 if you want to access from another host

DatabaseConf:
  Type: mysql # support mysql, postgres, sqlite3
  Host: "127.0.0.1" # change to your own mysql address
  Port: 3306
  DBName: simple_admin # database name, you can set your own name
  Username: root # username
  Password: "123456" # password
  MaxOpenConn: 100 # the maximum number of opened connections in the  connection pool
  SSLMode: disable # in postgresql, disable or require
# DBPath: /home/data/sqlite.db # Database Path, you must set it when you use sqlite3.

# casbin rule
CasbinConf:
  ModelText: |
    [request_definition]
    r = sub, obj, act
    [policy_definition]
    p = sub, obj, act
    [role_definition]
    g = _, _
    [policy_effect]
    e = some(where (p.eft == allow))
    [matchers]
    m = r.sub == p.sub && keyMatch2(r.obj,p.obj) && r.act == p.act

Log:
  ServiceName: coreApiLogger
  Mode: file
  Path: /home/ryan/logs/core/api # log saving path，use filebeat to sync
  Level: info
  Compress: false
  KeepDays: 7 # Save time (Day)
  StackCoolDownMillis: 100

RedisConf:
  Host: 127.0.0.1:6379 # Change to your own redis address
  Type: node
  # Pass: xxx  # You can also set the password
```

### Sync dependencies

```shell
go mod tidy
```

### Run rpc service

```bash
cd rpc

go run core.go -f etc/core_dev.yaml
```

### Run api service

```bash
cd api

go run core.go -f etc/core_dev.yaml
```

## Frontend setting

### Clone the code

```shell
git clone https://github.com/suyuan32/simple-admin-backend-ui.git
```

### Sync dependencies

```shell
pnpm install
```

::: warning
If downloading dependencies fails, please upgrade pnpm to the latest version
:::

### Run in development mode

```shell
pnpm serve
```

### Build

```shell
pnpm build
```

### Preview

```shell
# build and preview
pnpm preview

# preview existing directory
pnpm preview:dist
```

> Notice: Set the API address

> .env.development

```text
# Whether to open mock
VITE_USE_MOCK = false

# public path
VITE_PUBLIC_PATH = /

# Cross-domain proxy, you can configure multiple
# Please note that no line breaks
VITE_PROXY = [["/sys-api","http://localhost:9100"],["/file-manager","http://localhost:9102"]]

VITE_BUILD_COMPRESS = 'none'

# Delete console
VITE_DROP_CONSOLE = false

# Basic interface address SPA
VITE_GLOB_API_URL=

# File upload address， optional
VITE_GLOB_UPLOAD_URL=/upload

# File store address
VITE_FILE_STORE_URL=http://localhost:8080

# Interface prefix
VITE_GLOB_API_URL_PREFIX=
```

> Mainly modify sys-api in VITE_PROXY， use localhost or 127.0.0.1 to connect local service，\
> you can also set your own address, file-manager is the API for upload it is optional

## Initialize database

::: warning
**_Important:_** You should create the database before initialization, the database name should be the same as core_dev.yaml.
:::

```shell
# visit the address
https://address:port/init

# default is
https://localhost:3100/init
```

> You can see

![pic](/assets/init_zh_cn.png)

> File manager service is optional [File Manager](/simple-admin/en/docs/file_manager.md)

::: warning
**After initialization, you should restart api and rpc service.**
:::
