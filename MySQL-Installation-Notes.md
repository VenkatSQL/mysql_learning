# MySQL Installation Methods and Notes

> Working notes for later review. Commands and package names below are distribution- and version-specific; verify them against the target OS and the MySQL release before running. The repository and bundle examples in the original notes use different Ubuntu releases (22.04 and 24.04), so do not mix their package commands.

## Installation methods at a glance

1. **Repository-based installation** — configure the MySQL APT/Yum repository, then install packages with the OS package manager. Internet access is normally required on the server (unless the repository is mirrored locally). The package manager resolves and installs dependencies and makes upgrades/removal easier.
2. **Package bundle installation** — download a bundle of OS packages, transfer it to the server, and install packages locally. This can work without server internet access, but package dependencies must also be available and dependency ordering can be difficult. A package manager may resolve dependencies only if their packages are available from configured sources or the bundle.
3. **Generic binary installation** — download and unpack a generic archive. This gives more control over paths and can support multiple instances, but you manually configure directories, permissions, initialization, client settings, and service management. Required system libraries must be present.

## 1. Repository-based installation

### What is the MySQL repository?

A MySQL software repository is a configured package source containing MySQL packages and metadata. After configuring it and refreshing the local package index, the package manager can find the available packages, select dependencies, install them, and later manage updates.

### Example: MySQL Community Server on Ubuntu

This example follows the Ubuntu APT workflow. Confirm the supported Ubuntu release and package version on the official MySQL download pages before using it.

```bash
sudo -i
cat /etc/*release                 # Identify the Linux distribution and release
 dpkg -l | grep -i mysql           # Check installed MySQL packages

# Download the MySQL APT repository configuration package.
# Use the current URL/version from the official MySQL Community Downloads page.
apt install -y gnupg
wget https://dev.mysql.com/get/mysql-apt-config_0.8.40-1_all.deb
dpkg -i mysql-apt-config_0.8.40-1_all.deb

# Refresh package metadata after adding or changing a repository.
apt-get update
apt search mysql-community
apt install mysql-community-server
```

The repository configuration package may prompt you to choose the MySQL product series and distribution. The exact menu options change over time. Installing `mysql-community-server` should ask APT to install its dependencies from configured package sources.

### Verify installation and service state

```bash
dpkg -l | grep -i mysql
ps -ef | grep -i '[m]ysqld'
service mysql status
systemctl status mysql
```

On a systemd installation, inspect the unit to see how the service is launched:

```bash
systemctl cat mysql
```

A typical unit may run `/usr/sbin/mysqld` and use a helper script for pre-start setup. Paths vary by package and release. Use the service manager rather than launching a second `mysqld` manually.

### Connect to the server

```bash
mysql -uroot -p
mysql -h <hostname> -u <username> -p   # Connect to a remote host
```

The root account may require a password or another configured authentication method. Do not put a real password directly in shell history or documentation. In the MySQL client, `\\s` displays connection status.

### `basedir` and `datadir`

Check the paths used by the running server:

```sql
SELECT @@basedir, @@datadir;
```

- **`basedir`** is the installation location for MySQL programs, libraries, support files, and related utilities. A package installation commonly places files under `/usr`.
- **`datadir`** holds database contents and server state. A package installation commonly uses `/var/lib/mysql`.

Do not manually remove files from either location. Deleting database files can cause data loss or prevent the server from starting. Back up data and use MySQL's supported administration procedures.

### `my.cnf` and configuration files

MySQL server options can be set in option files, commonly called `my.cnf`. On Ubuntu/Debian, `/etc/mysql/my.cnf` commonly includes additional configuration directories; Red Hat-family installations commonly use `/etc/my.cnf`. Actual paths and include directives depend on the package.

Inspect the active files and includes before editing:

```bash
less /etc/mysql/my.cnf
```

Options for the server generally belong under a `[mysqld]` group. Client options can be set under `[client]`. If the main file includes a directory, MySQL can read applicable `.cnf` files from that directory. Later option-file settings can override earlier settings, depending on option and include order.

For example, after placing an appropriate setting in the active server configuration, restart and verify it:

```ini
[mysqld]
max_connections=124
```

```bash
sudo systemctl restart mysql
mysql -uroot -p -e 'SELECT @@max_connections;'
```

Use the correct option name (`max_connections`, plural) and confirm which file is actually being read; do not assume an arbitrary similarly named directory is active.

### Startup script / service

The package normally installs and enables a service unit. Common operations are:

```bash
sudo systemctl start mysql
sudo systemctl stop mysql
sudo systemctl restart mysql
sudo systemctl status mysql
```

The service unit is the startup configuration; use `systemctl cat mysql` to inspect its full contents and `systemctl enable mysql` if it should start at boot. Service names and unit paths can differ across distributions.

## 2. Installing from a package bundle

### What is a bundle?

A MySQL package bundle is an archive containing several individual OS packages (for example, `.deb` files). It is useful when packages need to be transferred to a machine without direct internet access. It does not necessarily include every operating-system dependency, so those dependencies must also be provided or already installed.

### Example: Debian/Ubuntu bundle workflow

The following example is for a specific Ubuntu 24.04 package build. Select a bundle matching the server's OS release and architecture; do not use this package set on Ubuntu 22.04 or another distribution without checking compatibility.

```bash
sudo -i
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-server_8.4.13-1ubuntu24.04_amd64.deb-bundle.tar
tar -xvf mysql-server_8.4.13-1ubuntu24.04_amd64.deb-bundle.tar
```

The archive may contain packages such as `mysql-common`, the community client and server packages, and optional development or test packages. Install only the required components, following the release-specific MySQL instructions. Installing `.deb` files one-by-one with `dpkg -i` can fail when dependencies are missing.

If all required dependency packages are available through configured APT sources, `apt` can often resolve dependencies for local package files, for example:

```bash
sudo apt install ./mysql-common_*.deb ./mysql-community-client-plugins_*.deb
```

For an offline host, first collect the complete dependency set for the same OS release and architecture. Do not blindly copy the dependency list below: library package names and versions vary by Ubuntu release.

```text
Example package names seen in one release (verify before use):
libaio1, libmecab2, libnuma1, perl, psmisc
```

Potential difficulties include:

- missing or incompatible OS dependencies;
- installing packages in the wrong order when using `dpkg` directly;
- mismatched package architecture or distribution release;
- maintaining security updates and upgrades without a configured repository.

After installation, verify packages and the service as described in the repository-based section.

## 3. Generic binary installation

A generic Linux archive is a compressed tar archive containing MySQL binaries and support files. It is suitable when custom installation paths or multiple independent instances are needed. It does not configure the operating system service or resolve all system-library dependencies for you.

### Check prerequisites and download

Choose the archive for the target operating system, CPU architecture, and compatible system libraries from the official MySQL Community Downloads page. The following URL is an example for MySQL 8.4 on x86-64 Linux:

```bash
# On a RHEL-family distribution, use the package manager appropriate to the release.
sudo dnf install -y wget xz
wget https://dev.mysql.com/get/Downloads/MySQL-8.4/mysql-8.4.13-linux-glibc2.28-x86_64.tar.xz
ldd --version
tar -xJf mysql-8.4.13-linux-glibc2.28-x86_64.tar.xz
```

`ldd --version` reports the system's C library information; compare it with the archive requirements. `xz`/`xz-utils` provides support for extracting `.tar.xz` archives (package name varies by distribution). The required libraries also vary by OS release. For example, package names such as `libaio`, `libnuma`, and ncurses/tinfo libraries are distribution-specific; verify availability before installing.

### Create directories and configure the instance

This example uses `/mysql/base` for the unpacked program files and separate data and log directories:

```bash
sudo mkdir -p /mysql/base /mysql/data /mysql/logs
# From the extracted archive directory, copy its contents into /mysql/base.
sudo cp -a mysql-8.4.13-linux-glibc2.28-x86_64/. /mysql/base/
sudo useradd --system --shell /usr/sbin/nologin mysql
sudo chown -R mysql:mysql /mysql
sudo chmod 750 /mysql/data /mysql/logs
```

If the `mysql` user already exists, do not create it again. Confirm the no-login shell path on the target distribution. Restrict ownership and permissions so only the MySQL account and authorized administrators can access database files.

Create `/mysql/my.cnf`:

```ini
[client]
socket=/mysql/mysql.sock

[mysqld]
basedir=/mysql/base
datadir=/mysql/data
log-error=/mysql/logs/mysqld.log
socket=/mysql/mysql.sock
port=3307
user=mysql
```

Use a dedicated configuration file and pass its path explicitly. A symbolic link to `/etc/my.cnf` is optional and distribution-dependent; check whether that path is already used before creating one.

### Initialize and start

Initialize the data directory once, before the first server start:

```bash
sudo /mysql/base/bin/mysqld \
  --defaults-file=/mysql/my.cnf \
  --initialize \
  --user=mysql
```

Initialization creates the system tables and writes a temporary initial root password to the error log on versions/configurations that use this process. Protect the log and retrieve the temporary password locally; do not copy it into notes or commit it to the repository.

For a temporary foreground/manual test, start the server under the `mysql` account using the same configuration. For ongoing use, create a systemd unit rather than backgrounding `mysqld` in a shell. Confirm the server is ready using its log and process/service status.

Connect using the configured socket and change the temporary password interactively:

```bash
/mysql/base/bin/mysql --defaults-file=/mysql/my.cnf -uroot -p
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '<choose-a-strong-unique-password>';
```

The initial temporary password must be entered when prompted. Avoid putting a real password in a command, script, or version-controlled file.

### Client socket and connection checks

If the client attempts to use `/tmp/mysql.sock`, specify the configured socket or ensure the client option file is being read:

```bash
/mysql/base/bin/mysql --socket=/mysql/mysql.sock -uroot -p
```

A local socket connection is different from a TCP connection. For TCP, connect to the server host and configured port, and ensure the server is configured to listen on the intended interface and that firewall/network access is appropriately restricted.

Useful checks:

```bash
ps -ef | grep -i '[m]ysqld'
/mysql/base/bin/mysql --socket=/mysql/mysql.sock -uroot -p -e 'SELECT @@basedir, @@datadir, @@port;'
```

### Create a systemd service

A custom binary installation does not automatically provide the package-managed `mysql.service`. A unit can be created as `/etc/systemd/system/mysqld.service`, then loaded and enabled. Review paths, security settings, and shutdown behavior for the target system before using a unit. Example:

```ini
[Unit]
Description=MySQL Server
After=network.target

[Service]
Type=notify
User=mysql
Group=mysql
ExecStart=/mysql/base/bin/mysqld --defaults-file=/mysql/my.cnf
ExecStop=/mysql/base/bin/mysqladmin --defaults-file=/mysql/my.cnf --socket=/mysql/mysql.sock shutdown
LimitNOFILE=5000

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysqld
sudo systemctl status mysqld
```

Some MySQL builds may require a different `Type` or readiness configuration; consult the release documentation and logs if the unit does not start. Stop a running server gracefully with its service or `mysqladmin` rather than using `kill -9` except as a last resort.

If SELinux blocks access, inspect audit logs and apply an appropriate SELinux policy/context for the custom paths. Do not disable SELinux as a routine fix; changing to permissive mode weakens system protection and may not persist after reboot.

For convenience, the binary directory can be added to a user's `PATH`, or the client can be invoked by its full path. Add the configuration's socket under `[client]` as shown above. Avoid changing the global `/etc/my.cnf` symlink without checking existing configuration.

## Review checklist

- Confirm every download URL, MySQL version, OS release, architecture, and package name.
- Keep Ubuntu 22.04 repository instructions separate from Ubuntu 24.04 bundle examples.
- Verify the active option files and service unit rather than assuming paths.
- Protect the data directory, error log, and temporary root password.
- Use strong credentials and never commit real passwords.
- Prefer graceful service management and diagnose SELinux denials rather than disabling enforcement.
- Test backup, restart, and recovery procedures before using an installation for important data.
