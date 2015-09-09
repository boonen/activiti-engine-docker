# Introduction

Dockerfile to build an [Activiti](#http://www.activiti.org/) BPM container image.

Based on [Frank Wang's work](https://github.com/eternnoir/activiti) and [Colin Woodcock's work](https://github.com/cwoodcock-docker/activiti) this has been extended with support for Activiti 6.

## Versions
* Java: 8u60-jdk
* Tomcat: 8.0.26
* Activiti: 6.0.0.Beta1
* PostgreSQL driver: 9.4-1201.jdbc41 (needs >= 9.4 server)
* Mysql connector: 5.1.36

# Using
This image can be deployed with different types of database:

* PostgreSQL
  * Linked Container
  * Remote Server
* MySQL
  * Linked Container
  * Remote Server

Which one is dependent on the DB_TYPE variable which **must** be supplied when running the image.

**For production use it is recommended to use a remote database server.**

When linking containers, environment variables are shared into the target.  This means the database server's administrative account credentials are exposed.  If your DB instance has multiple schemas this could present a security risk.  Better to have a dedicated account that just gives the permissions needed.

## Accessing
Once deployed you can access the UI via:

```
http://<ip of docker host>:<container's 8080 port>/activiti-app
```

And the REST resources via:

```
http://<ip of docker host>:<container's 8080 port>/activiti-rest
```

Login with *kermit/kermit*.

### Ports
Some of the commands below use docker's `-P` flag which maps the exposed ports to random ports on the Docker host.  This may/may not be what you want.  You can use `-p 8080:nnnn` instead if you want it assigned to a specific port.

You can use `docker ps` to see what the mapping is.

<a id="using_postgres"></a>
## PostgresSQL

<a id="using_postgres_linked"></a>
### Linked Container
There is a simple PostgreSQL 9.4 image with an *activiti* database that you can use:

```
docker run --name bpmdb -e POSTGRES_PASSWORD=changeme -d cwoodcock/postgres-activiti
```

Change the password to be something more appropriate.

Now you can launch the Activiti server with:

```
docker run -p 8080:8080 --name activiti --link bpmdb:bpmdb -e DB_TYPE=postgres -e DB_PASS=changeme -d boonen/activiti-engine-docker
```

**It is important that the alias on the link (the second part) is set to bpmdb.**

You can name you database container anything you like though e.g. If you have named your database container *mmmpie* the command would look like:

```
docker run -P -d --name activiti --link mmmpie:bpmdb -e DB_TYPE=postgres boonen/activiti-engine-docker
```

<a id="using_postgres_remote"></a>
### Remote Server
You do not need to do this first step if you already have a server.  This is purely for demo/doco purposes.  First we'll launch a vanilla PostgreSQL 9.4 instance and then get a psql shell to it:

```
docker run --name postgres -e POSTGRES_PASSWORD=changeme -p 5432:5432 -d postgres:9.4
docker run -it --link postgres:postgres --rm postgres:9.4 sh -c 'exec psql -h "$POSTGRES_PORT_5432_TCP_ADDR" -p "$POSTGRES_PORT_5432_TCP_PORT" -U postgres'
```
Then log in with the POSTGRES_PASSWORD, in this case *changeme*.

Notice that the postgres container maps its exposed port onto the Docker host, making it accessible externally.

From now on it's exactly the same as if you had a pizza box.

At the `postgres=#` prompt

```
CREATE DATABASE activiti;
REVOKE CONNECT ON DATABASE activiti FROM PUBLIC;
CREATE ROLE activiti WITH LOGIN PASSWORD 'changeme';
GRANT ALL PRIVILEGES ON DATABASE activiti TO activiti;
\q
```

You can get fancier e.g. by precreating the DB and only granting usage rights to the app.  But you will need to talk to the beardy guy in the corner for that.

Now the DB is setup we can launch the Activiti image:

```
docker run -P -d --name activiti -e DB_TYPE=postgres -e DB_HOST=192.168.59.103 -e DB_USER=activiti -e DB_PASS=changeme boonen/activiti-engine-docker
```
\* *assumes the PostgreSQL server is listening on 192.168.59.103:5432*

<a id="using_mysql"></a>
## MySQL

<a id="using_mysql_linked"></a>
### Linked Container

```
docker run --name bpmdb -e MYSQL_ROOT_PASSWORD=changeme -d mysql:latest
docker run -it --link bpmdb:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p'
```

*Wait a few seconds before running the second command just to give the server time to startup.*

Enter the MYSQL\_ROOT\_PASSWORD, in this case *changeme*.

At the `mysql>` prompt:

```sql
CREATE DATABASE IF NOT EXISTS `activiti` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
exit
```

Now you can run the Activiti container:

```
docker run -P -d --name activiti --link bpmdb:bpmdb -e DB_TYPE=mysql cwoodcock/activiti
```

**It is important that the alias on the link (the second part) is set to bpmdb.**

You can name you database container anything you like though e.g. If you have named your database container *mmmpie* the command would look like:

```
docker run --name activiti --link mmmpie:bpmdb -e DB_TYPE=mysql -P -d boonen/activiti-engine-docker
```

### Remote Server
You do not need to do this first step if you already have a server.  This is purely for demo/doco purposes.  First we will launch an official MySQL container, then get a mysql shell to it:

```
docker run --name bpmdb -e MYSQL_ROOT_PASSWORD=changeme -p 3306:3306 -d mysql:latest
docker run -it --link bpmdb:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p'
```

*Wait a few seconds before running the second command just to give the server time to startup.*

Note the use of the `-p` flag to map the MySQL container's 3306 port onto the Docker host.

Enter the MYSQL\_ROOT\_PASSWORD, in this case *changeme*.

At the `mysql>` prompt:

```sql
CREATE USER 'activiti'@'%.%.%.%' IDENTIFIED BY 'changeme';
CREATE DATABASE IF NOT EXISTS `activiti` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT ALL PRIVILEGES ON `activiti`.* TO 'activiti'@'%.%.%.%';
exit
```

You could (should) replace the *%.%.%.%* wildcards to be more restrictive allowing only specific hosts to connect.

Now you can run the Activiti container:

```bash
docker run -d -P \
  --name=activiti \
  -e DB_TYPE=mysql \
  -e DB_HOST=192.168.59.103 \
  -e DB_USER=activiti \
  -e DB_PASS=changeme \
  cwoodcock/activiti
```
\* *assumes the MySQL server is listening on 192.168.59.103:3306*

# Advanced Configuration

*Please refer the docker run command options for the `--env-file` flag where you can specify all required environment variables in a single file. This will save you from writing a potentially long docker run command.*

Below is the complete list of available options that can be used to customize your Activiti installation.

## Required Parameters
- **DB_TYPE**: mysql or postgres

## Optional Parameters
- **DB_HOST**: The database server hostname.
- **DB_PORT**: The database server port.  Has sane defaults depending on DB_TYPE (3306 for mysql, 5432 for postgres).
- **DB_NAME**: The database name. Defaults to `activiti`.
- **DB_USER**: The database user. When linking, it uses the root user for the database otherwise `activiti`.
- **DB_PASS**: The database password.  When linking this will be discovered from the environment, when remote it **must** be supplied.
- **TOMCAT\_ADMIN\_USER**: Tomcat admin user name. Defaults to `admin`.
- **TOMCAT\_ADMIN\_PASSWORD**: Tomcat admin user password. Defaults to `admin`.

The initialisation script will attempt to discover DB_* parameters (other than DB\_TYPE) however if supplied they take precedence.

# References

* http://activiti.org/
* http://github.com/Activiti/Activiti
* http://tomcat.apache.org/
* http://dev.mysql.com/downloads/connector/j/5.1.html
* https://github.com/jpetazzo/nsenter
* https://jpetazzo.github.io/2014/03/23/lxc-attach-nsinit-nsenter-docker-0-9/
