# Integration TiDB with ProxySQL step by step

**_English_** | [中文](/step-by-step-integration-zh.md)

This document briefly describes how to integrate **_TiDB_** with **_ProxySQL_** using CentOS 7 as an example. If you have integration needs for other systems, check out the [Try Out](#4-try-out) section, which is an introduction to deploying a test integration environment using **_Docker_** and **_Docker Compose_**. You can also refer yourself to the following links for more information:

- [TiDB Documentation](https://docs.pingcap.com/)
- [TiDB Developer Guide](https://docs.pingcap.com/tidb/stable/dev-guide-overview)
- [ProxySQL Documentation](https://proxysql.com/documentation/)

## 1. Startup TiDB

### 1.1 Test Environment - Source compilation

1. Download [TiDB](https://github.com/pingcap/tidb) code, enter `tidb-server` folder and run `go build`.

    ```sh
    git clone git@github.com:pingcap/tidb.git
    cd tidb/tidb-server
    go build
    ```

2. Use the configuration file tidb-config.toml to start TiDB. Note that:
    - Use `unistore` as the storage engine, which is the test storage engine for TiDB, so please use it for testing only.
    - `TIDB_SERVER_PATH`: The location of the binary compiled with `go build` in the previous step. For example, you did the previous step under `/usr/local`, then `TIDB_SERVER_PATH` should be: `/usr/local/tidb/tidb-server/tidb-server`.
    - `LOCAL_TIDB_LOG`: The location to export TiDB logs

    ```sh
    ${TIDB_SERVER_PATH} -config ./tidb-config.toml -store unistore -path "" -lease 0s > ${LOCAL_TIDB_LOG} 2>&1 &
    ```

### 1.2 Test Environment - TiUP startup

[TiUP](https://docs.pingcap.com/tidb/stable/tiup-overview), as the package manager, makes it far easier to manage different cluster components in the TiDB ecosystem.

1. Install TiUP

    ```sh
    curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
    ```

2. Test environment start TiDB

    ```sh
    tiup playground
    ```

### 1.3 Formal Environment

The formal environment is much more complex than the test environment, so we recommend reading the article [Deploy a TiDB Cluster Using TiUP](https://docs.pingcap.com/tidb/stable/production-deployment-using-tiup) and then deploying it based on hardware conditions.

## 2. Startup ProxySQL

### 2.1 Install by YUM

1. Adding repository:

    ```sh
    cat <<EOF | tee /etc/yum.repos.d/proxysql.repo
    [proxysql_repo]
    name= ProxySQL YUM repository
    baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.1.x/centos/\$releasever
    gpgcheck=1
    gpgkey=https://repo.proxysql.com/ProxySQL/repo_pub_key
    EOF
    ```

2. Install:

    ```sh
    yum install proxysql
    ```

3. Startup:

    ```sh
    systemctl start proxysql
    ```

### 2.2 Other

Please read [ProxySQL Github page](https://github.com/sysown/proxysql#installation) or the [official documentation](https://proxysql.com/documentation/) for installation.

## 3. ProxySQL Configuration

We need to write the host of TiDB in the ProxySQL configuration to use it as a proxy for TiDB. The required configuration items are listed below and the rest of the configuration items can be found in the ProxySQL [official documentation](https://proxysql.com/documentation/).

### 3.1 Simple Introduction to ProxySQL Configuration

ProxySQL uses a separate port for configuration management and another port for proxying. We call the entry point for configuration management **_ProxySQL Admin interface_** and the entry point for proxying **_ProxySQL Proxy interface_**.

- **_ProxySQL Admin interface_**: 读写权限用户仅可本地登录，无法开放远程登录。只读权限用户可远程登录。默认端口 `6032`。默认读写权限用户名 `admin`，密码 `admin`。默认只读权限用户名 `radmin`，密码 `radmin`。
- **_ProxySQL Proxy interface_**: 用于代理，将 SQL 转发到配置的服务中。

- **_ProxySQL Admin interface_**: Read-write access users can log in locally only. Read-only users can log in remotely. The default port is `6032`, default read-write user name is `admin`, password is `admin`. And default read-only user name is `radmin`, password is `radmin`.
- **_ProxySQL Proxy interface_**: Used as a proxy to forward SQL to the configured service.

![proxysql config flow](/doc-assert/proxysql_config_flow.png)

ProxySQL has three layers of configuration: `runtime`, `memory`, and `disk`. You can change the configuration of the `memory` layer only. After changing the configuration, you can use `load xxx to runtime` to make the configuration effective, or you can use `save xxx to disk` to save to the disk to prevent data loss.

![proxysql config layer](/doc-assert/proxysql_config_layer.png)

### 3.2 Configure TiDB Server

Add TiDB backend in ProxySQL, you can add multiple TiDB backends if you have more than one. Please do this at **_ProxySQL Admin interface_**:

```sql
insert into mysql_servers(hostgroup_id,hostname,port) values(0,'127.0.0.1',4000);
load mysql servers to runtime;
save mysql servers to disk;
```

Field Explanation:

- `hostgroup_id`: ProxySQL manages backend services by **hostgroup**, you can configure several services that need load balancing as the same hostgroup, so that ProxySQL will distribute SQL to these services evenly. And when you need to distinguish different backend services (such as read/write splitting scenario), you can configure them as different hostgroups.
- `hostname`: IP or domain of the backend service.
- `port`: The port of the backend service.

### 3.3 Configure Proxy Login User

Add a TiDB backend login user to ProxySQL. ProxySQL will allow this account to login **_ProxySQL Proxy interface_** and ProxySQL will use it to create a connection to TiDB, so make sure this account has the appropriate permissions in TiDB. Please do this at **_ProxySQL Admin interface_**:

```sql
insert into mysql_users(username,password,active,default_hostgroup,transaction_persistent) values('root','',1,0,1);
load mysql users to runtime;
save mysql users to disk;
```

Field Explanation:

- `username`: username
- `password`: password
- `active`: `1` is active, `0` is inactive, only the `active = 1` user can login.
- `default_hostgroup`: This user default **hostgroup**.
- `transaction_persistent`: A value of `1` indicates transaction persistence, i.e., when a connection opens a transaction using this user, then until the transaction is committed or rolled back, 
All statements are routed to the same **hostgroup**.

### 3.4 Configure by Config File

In addition to configuration using **_ProxySQL Admin interface_**, configuration files can also be used for configuration. In [Official Explanation](https://github.com/sysown/proxysql#configuring-proxysql-through-the-config-file), the configuration file should only be considered as a secondary way of initialization and not as the primary way of configuration. The configuration file is only read when the SQLite database is not created and the configuration file will not continue to be read subsequently. Therefore, when using the config file, you should delete the SQLite database. It will ***LOSE*** the changes you made to the configuration in **_ProxySQL Admin interface_**:

```sh
rm /var/lib/proxysql/proxysql.db
```

The config file is located at `/etc/proxysql.cnf`, we will translate the above required configuration to the config file way, only change `mysql_servers`, `mysql_users` two nodes, the rest of the configuration items can check `/etc/proxysql.cnf`: `/etc/proxysql.cnf`.

```
mysql_servers =
 (
 	{
 		address="127.0.0.1"
 		port=4000
 		hostgroup=0
 		max_connections=2000
 	}
 )

mysql_users:
 (
    {
 		username = "root"
        password = ""
 		default_hostgroup = 0
 		max_connections = 1000
 		default_schema = "test"
 		active = 1
		transaction_persistent = 1
 	}
 )
```

Then use `systemctl restart proxysql` to restart the service and it will take effect. The SQLite database will be created automatically after the confige file takes effect and the confige file will not be read again.

### 3.5 Other Config Items

The above config items are required. You can get all the config items name and their roles in the [Global Variables](https://proxysql.com/documentation/global-variables/) article in the ProxySQL documentation.
## 4. Try Out

You can use Docker and Docker Compose for quick start. Make sure the ports `4000` and `6033` are free.

```sh
docker-compose up -d
```

This has completed the startup of an integrated TiDB and ProxySQL environment, which will start two containers. ***DO NOT*** use it to create  in a production environment. You can connect to the port `6033` (ProxySQL) using the username `root` and an empty password. The container specific configuration can be found in [docker-compose.yaml](/docker-compose.yaml) and the ProxySQL specific configuration can be found in [proxysql-docker.cnf](/proxysql-docker.cnf).

## 5. Use

### 5.1 MySQL Client Connect ProxySQL

Run:

```sh
mysql -u root -h 127.0.0.1 -P 6033 -e "SELECT VERSION()"
```

Result:

```sql
+--------------------+
| VERSION()          |
+--------------------+
| 5.7.25-TiDB-v6.1.0 |
+--------------------+
```

### 5.2 Run Integration Test

If you satisfy the following dependencies:

- Permissions for the [tidb-test](https://github.com/pingcap/tidb-test) code repository tidb-test
- The machine needs to be connected to the network
- Golang SDK
- Git
- One of the following:

    1. Local Startup

        - CentOS 7 machine (can be a physical or virtual machine)
        - Yum

    2. Docker Startup

        - Docker
        - Docker Compose

Then you can run locally: `. /proxysql-initial.sh && . /test-local.sh` or use Docker: `. /test-docker.sh` to run integration tests.

There is more information available in the [integration test documentation](/README.md).
