# MySQL Installation

Learning notes comparing repository-based, package-bundle, and generic binary installations of MySQL, with example commands for Ubuntu/Debian and RHEL-family systems.

> **Version and platform note:** Commands below come from example labs using Ubuntu 22.04, Ubuntu 24.04 packages, and a RHEL-family server. Package names, dependency versions, download URLs, and paths vary by operating-system release and MySQL version. Check the current MySQL documentation before using these commands in a real environment.
>
> **Credential note:** Never use the example password `root@1234` in a real installation. Set a unique, strong password and protect it as a secret.

## Three installation methods

| Method | Internet needed during installation? | Main characteristics |
| --- | --- | --- |
| Repository-based | Usually yes | Easy to manage; the package manager resolves and installs dependencies. Configure the MySQL repository, then install only the packages needed. |
| Manual package / bundle | Not necessarily; packages must first be downloaded and transferred | A bundle can be installed without internet on the target host, but dependencies and package installation order must be handled manually. |
| Generic binary | Not necessarily; the archive must first be downloaded and transferred | Download and configure binaries manually, including dependencies, base directory, data directory, logs, configuration, and startup service. Allows multiple instances to be laid out independently. |

## Concepts to understand

### Repository-based installation

A repository is a configured source of software packages and package metadata. After configuring the MySQL repository and refreshing the local package index, the package manager can find MySQL packages and their dependencies. Install the package or packages required for the server; the package manager resolves and installs dependent packages.

### `basedir`

The MySQL base directory contains the MySQL program files: libraries, binaries, scripts, utilities, and related files needed to install and run the server. A repository installation in the Ubuntu example reports `/usr/` as `@@basedir`.

### `datadir`

The data directory stores MySQL's database and server data files. In the Ubuntu example, it is `/var/lib/mysql`. The notes observe a directory for each database and table data files such as `EMP1.ibd`; exact on-disk layouts can vary by configuration and MySQL version. Do not remove files from the data directory. Removing base-directory files can often be remedied by reinstalling packages or binaries; deleting data files can destroy database data.

### `my.cnf`

`my.cnf` is a MySQL configuration file. On Ubuntu/Debian, the main configuration is commonly `/etc/mysql/my.cnf`; on Red Hat/CentOS-family systems it is commonly `/etc/my.cnf`. MySQL can read included configuration files as well. An `!includedir` directive causes configuration files in the named directory to be included. Server settings, including InnoDB and other server parameters, belong in the `[mysqld]` group. Client settings can be placed in `[client]`.

When the same option is set in multiple loaded files, the effective value depends on configuration loading order; inspect the active configuration and verify the result from MySQL rather than assuming which file wins.

### Startup script / service

A startup service manages starting, stopping, and restarting the MySQL server. Package installations normally install a systemd service. A generic binary installation may require creating a systemd unit manually. Use the actual service name for the installation (`mysql` in the Ubuntu package example and `mysqld` in the binary example).

## Repository-based installation (Ubuntu example)

### Configure the MySQL APT repository and install the server

```bash
sudo su                         # Switch to the root account.
cat /etc/*release               # Check the Linux distribution and version; lab: Ubuntu 22.04.
dpkg -l | grep -i mysql         # List installed MySQL packages.

# Select MySQL Community Server, the desired LTS release, and the Ubuntu server
# package on the MySQL Community downloads site. The lab used this download page:
# https://dev.mysql.com/downloads/file/?id=559063
# Copy the link address and download the repository configuration package:
wget https://dev.mysql.com/get/mysql-apt-config_0.8.40-1_all.deb

# Install the prerequisite, then configure the repository package:
apt install gnupg -y
dpkg -i mysql-apt-config_0.8.40-1_all.deb

# Refresh package metadata after adding/updating/removing a repository:
apt-get update

# Search for the Community packages and install the server:
apt search mysql-community
apt install mysql-community-server
```

The lab's `apt install mysql-community-server` output listed these additional/new packages (11 newly installed, 0 upgraded, 0 removed, and 29 not upgraded):

- `libmecab2`
- `mecab-ipadic`
- `mecab-ipadic-utf8`
- `mecab-utils`
- `mysql-client`
- `mysql-common`
- `mysql-community-client`
- `mysql-community-client-core`
- `mysql-community-client-plugins`
- `mysql-community-server`
- `mysql-community-server-core`

This demonstrates the repository-based advantage: the package manager finds and installs dependencies.

### Verify installation and service status

```bash
dpkg -l | grep -i mysql         # List MySQL packages.
ps -ef | grep -i mysql          # Check processes; mysqld is the server daemon.
service mysql status
systemctl status mysql
```

The lab's `systemctl status mysql` output identified the service unit and key settings similar to:

```text
Loaded: loaded (/usr/lib/systemd/system/mysql.service; enabled; preset: enabled)
User=mysql
Group=mysql
ExecStartPre=+/usr/share/mysql-8.4/mysql-systemd-start pre
ExecStart=/usr/sbin/mysqld
```

The service's startup commands are responsible for starting MySQL; the package-managed service also supports stopping and restarting it.

### Connect to MySQL

```bash
mysql
```

The lab initially returned:

```text
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```

Connect by prompting for the root password instead:

```bash
mysql -uroot -p
```

Connect to a remote host with:

```bash
mysql -h<hostname> -u<username> -p
```

In the MySQL client, inspect the current connection and server paths:

```sql
\s
SELECT @@basedir, @@datadir;
```

The lab reported `basedir` as `/usr/` and `datadir` as `/var/lib/mysql`.

### Inspect common installation paths

```bash
cd /usr/                       # Example basedir from the lab.
ls bin/mysql*
ls lib/mysql/
ls share/
ls share/mysql-8.4/

cd /var/lib/mysql               # Example datadir from the lab.
ls
```

The base directory contains libraries, binaries, scripts, and utilities used to run and install MySQL. The data directory contains database files. Do not remove files from either directory; in particular, removing data files can cause data loss.

### Configuration files and option precedence

On Ubuntu/Debian, inspect the main configuration file and included directories:

```bash
less /etc/mysql/my.cnf
```

The lab notes that `!includedir` is used so MySQL also reads `.cnf` files from the included directory. On Red Hat/CentOS-family Linux, the common main configuration path is `/etc/my.cnf`.

Server-wide configuration belongs under `[mysqld]`. For example, the lab inspected `max_connections`:

```sql
SELECT @@max_connections;
```

The notes describe setting `max_connections=100` in `/etc/mysql/my.cnf` under `[mysqld]`, and `max_connections=124` in an included server configuration file. The effective value observed after restart was `124`, because the later-loaded child configuration supplied the effective value in that setup.

Example verification flow from the notes:

```bash
# Inspect the main configuration and the included server configuration files.
less /etc/mysql/my.cnf
less /etc/mysql/mysql.conf.d/mysqld.cnf

# Restart the service after changing configuration.
service mysql restart

# Reconnect to MySQL if needed, then verify the effective value:
mysql -uroot -p
```

```sql
\\r
SELECT @@max_connections;
```

Expected value in the lab:

```text
124
```

> The notes also mention `/etc/mysql/mysqld.conf.d/mysqld.cnf`; Ubuntu package layouts commonly use `/etc/mysql/mysql.conf.d/`. Check the actual `!includedir` paths on the host before editing a file.

## Manual package installation (MySQL bundle)

A package bundle is an archive containing multiple `.deb` packages. Download the bundle, transfer it to the target server if needed, and extract it. This approach can support an offline target host, but all required dependencies must be available and installed. Packages may need to be installed in a particular order; installing them manually can produce dependency errors and is more difficult than repository-based installation.

### Download and extract the bundle

The example uses a MySQL 8.4.13 Ubuntu 24.04 bundle:

```bash
sudo su
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-server_8.4.13-1ubuntu24.04_amd64.deb-bundle.tar
tar -xvf mysql-server_8.4.13-1ubuntu24.04_amd64.deb-bundle.tar
```

The archive in the lab contained these packages:

```text
libmysqlclient24_8.4.13-1ubuntu24.04_amd64.deb
libmysqlclient-dev_8.4.13-1ubuntu24.04_amd64.deb
mysql-client_8.4.13-1ubuntu24.04_amd64.deb
mysql-common_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-client_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-client-core_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-client-plugins_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-server_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-server-core_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-server-debug_8.4.13-1ubuntu24.04_amd64.deb
mysql-community-test_8.4.13-1ubuntu24.04_amd64.deb
mysql-server_8.4.13-1ubuntu24.04_amd64.deb
mysql-testsuite_8.4.13-1ubuntu24.04_amd64.deb
```

### Install packages and dependencies

The lab installs the client packages in this order, then installs dependencies and the server packages. Versions and dependency names differ by MySQL and Ubuntu release; verify them for the target system.

```bash
dpkg -i mysql-common_8.4.13-1ubuntu24.04_amd64.deb
dpkg -i mysql-community-client-plugins_8.4.13-1ubuntu24.04_amd64.deb
dpkg -i mysql-community-client-core_8.4.13-1ubuntu24.04_amd64.deb
dpkg -i mysql-community-client_8.4.13-1ubuntu24.04_amd64.deb
dpkg -i mysql-client_8.4.13-1ubuntu24.04_amd64.deb

# Example dependency command from the lab. Package names vary by OS release.
apt install libaio1 libmecab2 libnuma1 perl psmisc perl-modules-5.34 libperl5.34 libgdbm-compat4 libgdbm6

dpkg -i mysql-community-server-core_8.4.13-1ubuntu24.04_amd64.deb
dpkg -i mysql-community-server_8.4.13-1ubuntu24.04_amd64.deb

# Verify installed packages:
dpkg -l | grep -i mysql
```

**Bundle-installation difficulty:** Unlike repository installation, the administrator must obtain the packages, satisfy dependencies, and handle installation ordering. This method is useful when the target has no internet connection, provided the bundle and dependencies have been downloaded and made available beforehand.

## Generic binary installation

Choose the generic Linux binary distribution when using this installation method. The example uses a compressed TAR archive. A binary installation requires manual configuration of the base directory, data directory, log files, `my.cnf`, dependencies, and startup service. It can be useful for keeping multiple MySQL instances in separate layouts on one server; the notes contrast this with the repository or bundle installations used in the lab.

A screenshot was referenced in the original notes as `image-20261109-052806.png`; the image itself was not included in the provided content.

### Binary installation on a RHEL-family system

The notes mention that `dnf` is used on newer RHEL releases and that it replaced `yum` from RHEL 8 onward. Install the archive utility and download the binary archive:

```bash
dnf install wget
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.13-linux-glibc2.28-x86_64.tar.xz
dnf update

# Extract the downloaded archive:
tar -xvf mysql-8.4.13-linux-glibc2.28-x86_64.tar.xz
cd mysql-8.4.13-linux-glibc2.28-x86_64
ls
```

The extracted directory in the lab contained:

```text
LICENSE  README  bin  docs  include  lib  man  share  support-files
```

The original notes also list these environment preparation commands for binary installations:

```bash
ldd --version
apt install xz-utils
useradd -r -s /bin/false mysql
apt install -y libaio1 libnuma1 libncurses5 libtinfo5
ln -sf /mysql/my.cnf /etc/my.cnf
```

These `apt` commands are for Debian/Ubuntu-family systems, while the archive example below uses `dnf` on a RHEL-family server. Use the package manager and dependency package names appropriate to the target distribution. Create the `mysql` system account for the binary installation; package-based installations in the notes create the account automatically.

### Create directories and configure MySQL

Create separate directories for the base files, data, and logs, locate the extracted folder if needed, and move its contents into the base directory:

```bash
mkdir -p /mysql/base /mysql/data /mysql/logs

# Find the extracted directory if its location is unknown:
sudo find / -type d -name 'mysql-8.4.13-linux-glibc2.28-x86_64'

# Example move from the lab (adjust the source path as needed):
mv /home/ec2-user/mysql-8.4.13-linux-glibc2.28-x86_64/* /mysql/base/
```

Create `/mysql/my.cnf` with the paths and server settings:

```ini
[client]
socket=/mysql/mysql.sock

[mysqld]
basedir = /mysql/base
datadir = /mysql/data
log-error = /mysql/logs/mysqld.log
socket = /mysql/mysql.sock
port = 3307
```

Create the MySQL system user if it does not already exist, set ownership, and restrict directory permissions:

```bash
useradd -r -s /bin/false mysql
chown -R mysql:mysql /mysql
chmod 750 /mysql/data /mysql/logs
```

The original notes use `chmod 750 data logs` while in `/mysql`; the absolute-path version above has the same intended effect. Permission `750` means the owner has read/write/execute, the group has read/execute, and others have no permissions.

### Initialize the data directory and start the server

The server needs initialized data files before it can start. The lab's data directory was empty before initialization.

```bash
/mysql/base/bin/mysqld --defaults-file=/mysql/my.cnf --initialize --user=mysql
```

This creates initial files in the configured data directory and writes the temporary root password to the configured error log. Validate the directories:

```bash
ls /mysql/data/
ls /mysql/logs/
```

Start the server in the background for the lab:

```bash
/mysql/base/bin/mysqld --defaults-file=/mysql/my.cnf --user=mysql &
```

Check the process:

```bash
ps -ef | grep mysql
```

The lab showed a `mysqld` process running as the `mysql` user from `/mysql/base/bin/mysqld` with `/mysql/my.cnf` as its defaults file.

### Connect using the configured socket and set the initial password

The MySQL client binary is `/mysql/base/bin/mysql`. A connection attempt without specifying the configured socket can fail because the client may try `/tmp/mysql.sock` instead:

```bash
/mysql/base/bin/mysql -p
```

Example error:

```text
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)
```

Connect through the socket configured in `my.cnf`:

```bash
/mysql/base/bin/mysql -p -S /mysql/mysql.sock
```

The first attempt in the lab returned access denied because the supplied password was not the temporary initial password. Find the temporary root password in the error log, then retry:

```bash
less /mysql/logs/mysqld.log
/mysql/base/bin/mysql -p -S /mysql/mysql.sock
```

After connecting, MySQL requires changing the temporary password before normal statements can be run. The lab used this example command; choose a unique, strong password instead of copying the example:

```sql
ALTER USER root@localhost IDENTIFIED BY 'root@1234';
```

The lab's connection status (`\s`) showed MySQL Community Server 8.4.13, a local UNIX socket connection, and `/mysql/mysql.sock` as the socket path. The notes explain that local socket connections use the socket, while remote connections use a TCP port such as the configured `3307` (or the usual `3306`, if configured).

### Configure a systemd startup service

A generic binary installation does not necessarily include a `mysql.service`. The lab found that `systemctl status mysql` returned “Unit mysql.service could not be found.” It created `/etc/systemd/system/mysqld.service` instead:

```ini
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=simple
User=mysql
Group=mysql
ExecStart=/mysql/base/bin/mysqld --defaults-file=/mysql/my.cnf
ExecStop=/mysql/base/bin/mysqladmin --defaults-file=/mysql/my.cnf shutdown
LimitNOFILE=5000

[Install]
WantedBy=multi-user.target
```

Reload systemd after creating or changing the unit:

```bash
systemctl daemon-reload
```

The notes mention checking for an existing MySQL process and stopping it before starting the service. They show `kill -9 42903` as a lab command. **Avoid `kill -9` as a routine shutdown method**: it forcibly terminates the process and may prevent a clean shutdown. Prefer the configured service's stop operation or `mysqladmin shutdown`, and first verify the process ID belongs to the intended server.

The lab then encountered `'/mysql/base/bin/mysqld': Permission denied` when checking the service. SELinux was found to be enforcing. The notes show these diagnostic commands and a temporary switch to permissive mode:

```bash
sestatus
getenforce
setenforce 0
getenforce
```

The output changed from `Enforcing` to `Permissive`; `sestatus` still showed SELinux enabled and the configured mode as enforcing. **This is a troubleshooting observation, not a recommended permanent fix.** Disabling enforcement can reduce system security. Investigate SELinux denials and configure appropriate labels or policy for the MySQL paths instead; any temporary mode change should be approved and restored according to the host's security policy.

### Add the binary directory to `PATH` and configure the client socket

The lab added the binary directory to root's Bash profile:

```bash
echo 'export PATH=$PATH:/mysql/base/bin' >> ~/.bash_profile
source ~/.bash_profile
```

The client socket was configured in `/mysql/my.cnf` under `[client]`, as shown earlier. The notes also create a symbolic link so `/etc/my.cnf` points to the same file:

```bash
ln -sf /mysql/my.cnf /etc/my.cnf
```

Once the binary path and socket configuration are set, the notes indicate that the MySQL client can be invoked without specifying the full client binary path and can use the configured socket. For example:

```bash
mysql -p
```

For an explicit local socket connection:

```bash
mysql -p -S /mysql/mysql.sock
```

## Quick comparison / recap

- **Repository-based:** Configure the repository, refresh package metadata, and install the needed package. The package manager resolves dependencies and normally configures the service and MySQL user.
- **Manual package bundle:** Download and extract the bundle, then install packages and dependencies in a compatible order. Useful when the target host has no internet, but more difficult to manage manually.
- **Generic binary:** Download and extract the generic Linux archive. Manually create the MySQL user, configure dependencies and directories, initialize data, set up credentials, configure the socket, and create a startup service. Provides control over paths and can support multiple independent instances.

## References from the notes

- MySQL Community downloads: https://dev.mysql.com/downloads/
- MySQL APT repository package used in the example: https://dev.mysql.com/get/mysql-apt-config_0.8.40-1_all.deb
- MySQL 8.4.13 Ubuntu 24.04 package bundle used in the example: https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-server_8.4.13-1ubuntu24.04_amd64.deb-bundle.tar
- MySQL 8.4.13 generic Linux archive used in the example: https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.13-linux-glibc2.28-x86_64.tar.xz
- MySQL Community download page used in the repository-installation notes: https://dev.mysql.com/downloads/file/?id=559063
