
# DXSpider Upgrade Guide to the Latest Mojo Build

This guide is intended for SYSOPs who already have a DXSpider installation. You can upgrade manually or use the automatic update script.

The examples use `/spider` as the installation path and `sysop` as the usual user account. Always check the actual values on your system.

## Method 1. Manual upgrade

### 1. Install the required packages

On Debian, Ubuntu, Raspberry Pi OS and derived distributions, log in as an administrator:

```bash
sudo -i
apt-get update
apt-get install perl git rsync curl iproute2 gzip procps build-essential cpanminus libssl-dev zlib1g-dev
```

Install the available Perl packages. The following loop allows you to continue if a package is unavailable in your distribution (run it in Bash):

```bash
for pkg in libtimedate-perl libnet-telnet-perl libcurses-perl libdigest-sha-perl \
    libdata-dumper-simple-perl libjson-perl libmojolicious-perl \
    libdata-structure-util-perl libmath-round-perl libev-perl \
    libjson-xs-perl libnet-cidr-lite-perl libdbi-perl libdbd-sqlite3-perl \
    libhtml-parser-perl libio-socket-inet6-perl libarchive-zip-perl \
    libmailtools-perl libwww-perl libio-pty-perl libauthen-sasl-perl \
    libnet-smtp-ssl-perl libemail-sender-perl; do
    apt-get install -y "$pkg" || echo "Check package: $pkg"
done
```

If a module is unavailable through APT, use `cpanm`. This block checks all modules required by the updater and installs only those that are missing:

```bash
for module in EV Mojolicious JSON JSON::XS Data::Structure::Util Math::Round \
    Net::CIDR::Lite DBI DBD::SQLite Date::Parse HTML::Parser \
    IO::Socket::INET6 Archive::Zip Curses Mail::Internet Digest::SHA \
    LWP::UserAgent LWP::Parallel::UserAgent DB_File IO::Pty \
    Authen::SASL Net::SMTP::SSL Email::Sender; do
    perl -M"$module" -e 1 2>/dev/null || cpanm "$module" || break
done
```

Make sure you have Perl 5.16.1 or later and Mojolicious 7.26 or later:

```bash
perl -e 'die "Perl version is too old\n" if $] < 5.016001'
perl -MMojolicious -e 'Mojolicious->VERSION(7.26)'
```

If any installation or check fails, do not proceed until the issue has been resolved.

### 2. Configure the user database

Identify the owner account and the actual installation path:

```bash
readlink -f /spider
stat -c '%U:%G' /spider/perl/cluster.pl
```

Switch to the user account that runs DXSpider (usually `sysop`; replace the name if different):

```bash
su - sysop
cd /spider
pwd -P
nano /spider/local/DXVars.pm
```

Add the following line before the final `1;`, only if `$userdsn` is not already defined:

```perl
our $userdsn = "dbi:SQLite:dbname=$root/local_data/dxusers.db";
```

Save the file. Do not remove the final `1;` or duplicate the variable.

### 3. Update scheduled tasks

```bash
nano /spider/local_cmd/crontab
```

Replace the equivalent existing tasks with the following, where applicable. Do not duplicate tasks or remove other node-specific tasks, such as connections to other clusters.

Select only one of the two BadIP blocks according to the installed build. Remove or comment out the other block. For a static public IP, the `set/external_ip` scheduled task is optional.

```perl
# Update dynamic public IP
0,30 * * * * run_cmd('set/external_ip')

# Automatic compression debug and log files
10 0 * * * spawn("$main::root/perl/compression.pl")

# AutoUpdate badspotter badnode
10 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badspotter')
11 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badnode')
12 * * * * run_cmd('load/badnode')
12 * * * * run_cmd('load/badspotter')

# AutoUpdate BadIP (only if build < 935)
33 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.torexit')
34 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.torrelay')
35 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.global')
36 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip.curated')
40 * * * * run_cmd('load/badip')

# AutoUpdate BadIP (only if build > 935)
33 * * * * run_cmd('download http://dxspider.dxcluster.net/download/badip*')
40 * * * * run_cmd('load/badip')

# AutoUpdate USDB
0 1 * * * run_cmd('download https://ftp.w1nr.net/usdbraw.gz')
2 1 * * * spawn('/spider/perl/create_usdb.pl /spider/local_data/usdbraw.gz')
6 1 * * * run_cmd('load/usdb')
9 1 * * * spawn('rm -f /spider/local_data/usdbraw.gz')

# AutoUpdate Keps
0 2 * * * run_cmd('download http://www.amsat.org/amsat/ftp/keps/current/nasabare.txt')
2 2 * * * spawn('/spider/perl/convkeps.pl -p /spider/local_data/nasabare.txt')
6 2 * * * run_cmd('load/keps')
9 2 * * * spawn('rm -f /spider/local_data/nasabare.txt')

# AutoUpdate CTY.DAT
0 3 * * * run_cmd('download http://www.country-files.com/cty/cty.dat')
2 3 * * * run_cmd('download http://www.country-files.com/cty/wpxloc.raw')
6 3 * * * spawn("/spider/perl/create_prefix.pl")
9 3 * * * run_cmd("load/prefix")
```

### 4. Update the startup file

```bash
nano /spider/scripts/startup
```

Remove any lines beginning with the following expressions, regardless of their assigned values:

```text
set/var $main::maxconnect_user
set/var $main::localhost_alias_ipv4
set/var $main::localhost_alias_ipv6
set/var @main::localhost_names
```

Keep the rest of the file unchanged.

### 5. Upgrade to the latest Mojo build

Run Git as the installation owner, not as root (unless root is actually the DXSpider user). Change to the DXSpider directory, not the user's home directory:

```bash
cd /spider
pwd -P
git rev-parse --show-toplevel
git remote -v
```

Check that the repository root matches the actual installation. If it is not a Git repository, use the automatic method.

Fetch the `mojo` branch from Dirk's official repository:

```bash
git fetch git://scm.dxcluster.org/spider refs/heads/mojo
```

**Only if this fails**, use the alternative repository:

```bash
git fetch https://github.com/EA3CV/dxspider.git refs/heads/mojo
```

If both fail, do not proceed. After a successful fetch:

```bash
git log -1 --oneline FETCH_HEAD
```

Before replacing the code, stop the node as described in section 6 and make a complete backup of the installation while the node is stopped, outside the DXSpider directory tree. Make sure there is sufficient disk space and that the backup includes local data and configuration.

**WARNING:** the following commands discard local changes to tracked files and replace the local `mojo` branch with the fetched revision. Do not use `git clean`, as it could remove required local files. If local configuration or data files are tracked, preserve them in the backup and review their restoration before starting the node.

```bash
cd /spider
git reset --hard FETCH_HEAD && git checkout -f -B mojo FETCH_HEAD
```

Verify the result:

```bash
git branch --show-current
git log -1 --oneline
git status --short
```

The branch must be `mojo`. Do not proceed if Git reports an error. Tracked files must match the fetched revision; untracked local files are preserved except those that prevent tracked files from being written. Check that the changes made in sections 2, 3 and 4 are still present.

### 6. Restart and verify the node

#### Stop the node (before applying the code changes in section 5)

If DXSpider runs as a systemd service, use an account with administrator privileges:

```bash
sudo systemctl stop dxspider.service
sudo systemctl status dxspider.service
```

If DXSpider runs manually, access the DXSpider console with SYSOP privileges and execute:

```text
shut
```

Do not confuse `shut` with a Linux shell command. Make sure the node has stopped before continuing.

#### Check manual startup

Once the upgrade is complete, run the following as the DXSpider user:

```bash
cd /spider/perl
/spider/perl/cluster.pl
```

Watch the startup messages. If errors appear, resolve them before starting the service. If startup is successful, open another terminal, access the console (usually `/spider/perl/console.pl`) and execute `shut` to stop the test process.

#### Final startup

If you use systemd, run the following from an account with administrator privileges:

```bash
sudo systemctl start dxspider.service
sudo systemctl status dxspider.service
```

You can also use `sudo service dxspider start` if your installation supports that command.

If you start DXSpider manually, run the following as the DXSpider user:

```bash
cd /spider/perl
/spider/perl/cluster.pl
```

Access the console and check the version and build using `show/version`.

## Method 2. Automatic upgrade

As an alternative to the manual method, complete only sections 3 and 4 above and use the EA3CV updater, which installs dependencies, prepares a backup and upgrades to the `mojo` branch, trying Dirk's repository first and then GitHub.

Repository: https://github.com/EA3CV/dxspider_info

Before running it, determine the actual installation path:

```bash
readlink -f /spider
```

If the node runs manually, stop it using `shut` from the console. If it uses systemd and is running, the updater handles stopping it.

Log in as root and download DXSpider Manager into a new working directory:

```bash
sudo -i
cd /root
mkdir dxspider-manager
cd dxspider-manager
apt-get install wget
wget -O dxspider-manager.pl https://raw.githubusercontent.com/EA3CV/dxspider_info/main/dxspider-manager.pl
chmod a+x dxspider-manager.pl
./dxspider-manager.pl
```

If the working directory already exists, use a different name to avoid mixing old downloads. If you are already root, omit `sudo -i`.

When the script asks for the DXSpider installation directory, enter the actual path obtained earlier (for example `/home/sysop/spider`). Do not enter `/home/sysop` or the directory where you downloaded the updater.

Select the upgrade option for an existing installation and follow the on-screen instructions. Once finished, check that the node starts correctly and verify its version. Do not select a fresh installation.
