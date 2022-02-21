---
layout: post
title: "Building a Secure FTP Server"
tags: 
  - Linux
  - Security
  - Ubuntu
  - Ubuntu 20.04
  - FTP
  - FTPS
  - SFTP
comments: true
toc: true
---

<!--NOTICE--{::nomarkdown}<div class="post-notice"><div class="post-notice-content">
Foreword: Expect all links to open in a new page. Comments are open to questions and suggestions. This article is in progress.
</div></div>{:/nomarkdown}-->

Building a secure FTP server should begin with a clear understanding of the mechanisms involved. When we talk about an FTP server, this commonly involves three protocols:

1. [__FTP__](https://en.wikipedia.org/wiki/File_Transfer_Protocol){:target="_blank"} — File Transfer Protocol. This is your basic protocol for transferring files.
2. [__FTPS__](https://en.wikipedia.org/wiki/FTPS){:target="_blank"} — FTP over SSL, or FTP Secure. This is an extension to the basic FTP protocol, which adds support for [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security){:target="_blank"} (Transport Layer Security).
3. [__SFTP__](https://en.wikipedia.org/wiki/SSH_File_Transfer_Protocol){:target="_blank"} — While the acronym is similar, this is SSH File Transfer Protocol, and it operates unrelated to both FTP and FTPS.

Deciding which protocols work best for you, depends largely on your project. There should almost never be a need to support basic FTP. The exception might be if you needed a simple file transfer solution between servers that exist within a private local network. Otherwise, using FTP is akin to serving an application over HTTP instead of HTTPS.

Let's explore some of the key differences between each protocol.

| | :FTP: | :FTPS: | :SFTP: |
| - | - | - | - |
| Command Port | 21 | 990 | 22 |
| Data Port | 20 | 989 (active mode) — passive mode is user defined, but by default any port 0—65535 | 22 |
| Security | : Basic FTP doesn't encrypt any communication between the client and the server :  | : Command and data channels are encrypted only if the client issues the necessary [AUTH](https://datatracker.ietf.org/doc/html/rfc2228#section-3){:target="_blank"} and [PROT](https://datatracker.ietf.org/doc/html/rfc2228#section-3){:target="_blank"} commands : | : Relies on SSH for data encryption over the wire - commands and data are always encrypted : |
| Connections | : At least 2: one port to issue commands and a separate port for data : || : Only 1 is required (commands and data use the same connection) : |
| Pros | : Anonymous FTP access in browser, and slightly faster due to having no encryption overhead : | : Widely known and supported, with better support for server-to-server file transfers : | : Commands and data are always encrypted, and is backed by solid standards : |
| Cons | : Connection details and data is transmitted in clear text : | : Requires a secondary DATA channel, which makes it harder to use behind firewalls and NATs : | : Limited server-to-server options, dependent on environment and application : |

{::nomarkdown}
<div class="post-notice"><div class="post-notice-icon"><svg aria-hidden="true" focusable="false" data-prefix="fad" data-icon="info-circle" role="img" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512" class="fa-fw fa-2x svg-inline--fa fa-info-circle fa-w-16"><g class="fa-group"><path fill="#98b3c6" d="M256 8C119 8 8 119.08 8 256s111 248 248 248 248-111 248-248S393 8 256 8zm0 110a42 42 0 1 1-42 42 42 42 0 0 1 42-42zm56 254a12 12 0 0 1-12 12h-88a12 12 0 0 1-12-12v-24a12 12 0 0 1 12-12h12v-64h-12a12 12 0 0 1-12-12v-24a12 12 0 0 1 12-12h64a12 12 0 0 1 12 12v100h12a12 12 0 0 1 12 12z" class="fa-secondary"></path><path fill="#305977" d="M256 202a42 42 0 1 0-42-42 42 42 0 0 0 42 42zm44 134h-12V236a12 12 0 0 0-12-12h-64a12 12 0 0 0-12 12v24a12 12 0 0 0 12 12h12v64h-12a12 12 0 0 0-12 12v24a12 12 0 0 0 12 12h88a12 12 0 0 0 12-12v-24a12 12 0 0 0-12-12z" class="fa-primary"></path></g></svg></div><div class="post-notice-content">Tip: To use FTPS in <a href="https://filezilla-project.org/" target="_blank">FileZilla</a>, set the Encryption option to: <em>Require explicit FTP over TLS</em>.</div></div>
{:/nomarkdown}

### Implicit -vs- Explicit in FileZilla

It’s worth noting that explicit FTPS uses port 21, where implicit FTPS uses port 990. With <em>explicit</em> mode, clients initially connect to the standard FTP port (21), and then upgrades the connection into secure FTPS mode (990), by issuing an [AUTH](https://datatracker.ietf.org/doc/html/rfc2228#section-3){:target="_blank"} command. By comparison, with <em>implicit mode</em>, its assumed the connection is always encrypted from the beginning.

As described in this [FileZilla Wiki](https://wiki.filezilla-project.org/FTP_over_TLS#Explicit_vs_Implicit_FTPS){:target="_blank"}, Explicit mode is considered more modern. When also considering that most clients and software libraries assume port 21 as the default, Explicit is recommended over Implicit.

You can switch to implicit mode by listening on port 990 instead of 21, and enabling the [implicit_ssl](http://vsftpd.beasts.org/vsftpd_conf.html) option in `vsftpd.conf`:

{% highlight conf %}
listen_port=990
implicit_ssl=YES
{% endhighlight %}

## Getting Started

I'm going to show you how to build a secure FTP server with the following features:

- Support for both FTPS and SFTP to maximize integration options for third-parties.
- Virtual FTPS users with custom authentication using a Berkeley DB database.
- Jailed user environments, so that users cannot access any files or directories outside of their own dedicated folders.
- Shell-less SFTP users, so that users can only perform file transfer operations.
- A command script for easy user management.

You're going to need the following tools and software:

- A linux-based virtual machine — I've used Ubuntu 20.04.
- [__vsftpd__](https://help.ubuntu.com/community/vsftpd){:target="_blank"} (Very Secure FTP Daemon) — this is our FTP server software.
- [__db-util__](https://packages.ubuntu.com/focal/db-util){:target="_blank"} — [Berkeley DB](https://www.oracle.com/database/technologies/related/berkeleydb.html){:target="_blank"} database utilities (this will be for our user database)
- SSL certificate (if you plan on creating your own DNS record) — otherwise, we can generate a self-signed certificate.

## Step 1: Install `vsftpd` and `db-util`

{% highlight bash %}
sudo apt update
sudo apt install vsftpd db-util
{% endhighlight %}

## Step 2: Configure `vsftpd`

Open the configuration file:

{% highlight bash %}
sudo nano /etc/vsftpd.conf
{% endhighlight %}

and replace the entire contents with the configuration below. I've added comments describing what each of the options are for. If you're looking for additional information, I recommend the [Red Hat Documentation](https://web.mit.edu/rhel-doc/5/RHEL-5-manual/Deployment_Guide-en-US/s1-ftp-vsftpd-conf.html){:target="_blank"} on vsftpd, as well as Ubuntu's [community help wiki](https://help.ubuntu.com/community/vsftpd){:target="_blank"}.

Be sure to look at the `pasv_address`, `rsa_cert_file` and `rsa_private_key_file` options at the bottom. You will need to update these values related to your own server.

{% highlight conf %}
# run vsftpd in stand-alone mode
listen=YES
listen_port=21

# disable IPv6 (cannot be used with stand-alone mode)
listen_ipv6=NO

# enable connection level logging (helps with troubleshooting)
# default log location: /var/log/vsftpd.log
xferlog_enable=YES
xferlog_std_format=NO
log_ftp_protocol=YES

# disable anonymous access
anonymous_enable=NO

# enable local accounts
local_enable=YES

# enable virtual users
guest_enable=YES
guest_username=ftp

# default umask for local users
# 022 allows only our local user to write, but anyone can read
# 077 is completely private, no other user can read or write
local_umask=077

# virtual users will have the same privileges
# as our local guest account
virtual_use_local_privs=YES

# write permissions for users
write_enable=YES
allow_writeable_chroot=YES

# virtual user directory
local_root=/home/vftp/$USER

# automatically generate a home directory for each virtual user
user_sub_token=$USER

# jail all users by default
chroot_local_user=YES
chroot_list_enable=NO
secure_chroot_dir=/var/run/vsftpd/empty

# hide info about file owner (user and group)
hide_ids=YES

# miscellaneous options
dirmessage_enable=YES
use_localtime=YES

# allow active mode connections
port_enable=YES
connect_from_port_20=YES
ftp_data_port=20

# use the virtual PAM service
pam_service_name=vsftpd.virtual

# set max connections and idle timeout
# to help against DoS attacks
max_per_ip=3
idle_session_timeout=300

# passive mode configuration
# enter your own servers public IP address
pasv_address=0.0.0.0
pasv_min_port=50000
pasv_max_port=50999
pasv_promiscuous=YES
pasv_enable=YES

# SSL
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
rsa_cert_file=/path/to/bundle.crt
rsa_private_key_file=/path/to/cert.key
{% endhighlight %}

## Step 3: Configuring firewalls

In passive mode, the client initiates a [PASV](https://datatracker.ietf.org/doc/html/rfc2228#section-3){:target="_blank"} command to the server, which requests an available port for data transmission. By default, vsftpd does not limit the port range, meaning a client could be returned a port anywhere between 0—65535.

Instead, we've defined a data port range between 50000—50999 in our `vsftpd.conf`, which gives us 999 available data ports for clients. If you anticipate having significant numbers of concurrent users, consider increasing this range.

Now that we've defined an explicit range, we need to allow this range of ports in our firewall. If you're using [Microsoft Azure](https://portal.azure.com/){:target="_blank"}, apply these rules to a Network Security Group that covers your FTP server. If you're using Ubuntu's [Uncomplicated Firewall (UFW)](https://help.ubuntu.com/community/UFW){:target="_blank"}, first make sure its enabled:

{% highlight bash %}
sudo ufw status
{% endhighlight %}

enable it if you need to:

{% highlight bash %}
sudo ufw enable
{% endhighlight %}

then allow the TCP port range we defined in our `vsftp.conf` config:

{% highlight bash %}
sudo ufw allow 50000:50999/tcp
{% endhighlight %}

In addition, we must also allow default ports common to the FTPS and SFTP protocols. Again, if you're using Azure, apply these rules to a Network Security Group. Otherwise, update your Ubuntu firewall:

{% highlight bash %}
sudo ufw allow 20,21,22,989,990/tcp
{% endhighlight %}

Here is a recap of the ports we're allowing:

- `20` — FTP data channel
- `21` — FTP command channel
- `22` — SSH
- `989` — FTPS data channel (active mode)
- `990` — FTPS command channel
- `50000—50999` — FTPS data channels (passive mode)

## Step 4: Create a new PAM service

PAM (short for Pluggable Authentication Modules), is a powerful suite of libraries that allow us to dynamically authenticate users in a Linux-based system. We're going to use the [pam_userdb](https://www.man7.org/linux/man-pages/man8/pam_userdb.8.html){:target="_blank"} module which will allow us to authenticate against a DB database.

Create a new PAM file that will use our new database (you'll create the database in step 6):

{% highlight bash %}
sudo nano /etc/pam.d/vsftpd.virtual
{% endhighlight %}

and save the following:

{% highlight bash %}
#%PAM-1.0
auth       required     pam_userdb.so db=/etc/vsftpd/users
account    required     pam_userdb.so db=/etc/vsftpd/users
session    required     pam_loginuid.so
{% endhighlight %}

Note that the path to the database file should be specified without the `.db` suffix.

As an extra, `pam_userdb` allows us to define whether passwords stored in our user database, are encrypted, by passing an additional [crypt](https://www.man7.org/linux/man-pages/man8/pam_userdb.8.html#OPTIONS){:target="_blank"} option.

{% highlight bash %}
#%PAM-1.0
auth       required     pam_userdb.so db=/etc/vsftpd/users crypt=crypt
account    required     pam_userdb.so db=/etc/vsftpd/users crypt=crypt
session    required     pam_loginuid.so
{% endhighlight %}

It's important to know, if you choose this option, passwords must be stored in [crypt(3)](http://www.linux-pam.org/Linux-PAM-html/sag-pam_userdb.html) form. The `crypt()` function relies on the legacy DES (Data Encryption Standard) algorithm, which only supports a maximum password length of 8 characters. This is not often expressly pointed out, but is described in the related [manual](Source: https://man7.org/linux/man-pages/man3/crypt.3.html):

> By taking the lowest 7 bits of each of the first eight characters of the key, a 56-bit key is obtained.  This 56-bit key is used to encrypt repeatedly a constant string (usually a string consisting of all zeros).

## Step 5: Create service directories

We're going to need some directories where user content will live. It is important that user directories are created inside a parent directory, which will act as our jail. We'll also need a place to keep our virtual user database.

{% highlight bash %}
# directory for our virtual FTPS users
sudo mkdir /home/vftp

# directory for our local SFTP users
sudo mkdir /home/sftp

# directory to store our user database
sudo mkdir /etc/vsftpd
{% endhighlight %}

## Step 6: Create FTPS user database

Rather than creating local users for FTPS, we're going to create virtual users — a feature of vsftpd. Enabling virtual FTPS users will help enhance the security of our FTP server.

First create a plain text file:

{% highlight bash %}
sudo nano /etc/vsftpd/users.txt
{% endhighlight %}

then enter your usernames and passwords on alternating lines, as described in the [db_load](https://manpages.ubuntu.com/manpages/focal/en/man1/db_load.1.html){:target="_blank"} documentation:

> If the database to be created is of type Btree or Hash, or the keyword keys is specified as set, the input must be paired lines of text, where the first line of the pair is the key item, and the second line of the pair is its corresponding data item.

For example, we're going to create the users `batman` with password `bat!`, and `robin` with password `cave!`:

{% highlight bash %}
batman
bat!
robin
cave!
{% endhighlight %}

Next, we'll use [db_load](https://manpages.ubuntu.com/manpages/focal/en/man1/db_load.1.html){:target="_blank"} to generate our user database. This will take `users.txt` as input, and output `users.db`:

{% highlight bash %}
sudo db_load -T -t hash -f /etc/vsftpd/users.txt /etc/vsftpd/users.db
{% endhighlight %}

The arguments we're passing here are:

- `-T` — allows non-Berkeley DB applications to easily load text files into databases.
- `-t <method>` — specify the underlying access method (required when using `-T`). Here we're using the [Hash](https://docs.oracle.com/database/bdb181/html/gsg/CXX/accessmethods.html){:target="_blank"} access method, which is best suited for large data sets (ie: many users), where we aren't concerned about sequential access. The hash method is also more memory efficient, as we can typically access data with a single I/O operation (compared to [B-tree](https://docs.oracle.com/database/bdb181/html/gsg/CXX/accessmethods.html){:target="_blank"} for example).
- `-f <file>` — read from the specified input file.
- `<output>` — the last argument is our desired output DB file.

Once you've created your user database, update its permissions:

{% highlight bash %}
sudo chmod 0600 /etc/vsftpd/users.db
{% endhighlight %}

You should also delete `users.txt`, if you no longer need it:

{% highlight bash %}
sudo rm /etc/vsftpd/users.txt
{% endhighlight %}

## Step 7: Create user directories

Now we need directories for our new users:

{% highlight bash %}
sudo mkdir -p /home/vftp/{batman,robin}
sudo mkdir -p /home/sftp/{batman,robin}
{% endhighlight %}

## Step 8: Create SFTP users

Since SFTP users are local users, let's go ahead and create them — keeping in mind that these are not the same as our virtual FTPS users.

{% highlight bash %}
sudo adduser --shell /bin/false batman
sudo adduser --shell /bin/false robin
{% endhighlight %}

Here we're passing the `--shell <shell>` argument, which defines what shell is loaded for the user on login. In this case, we're supplying `/bin/false`, which is actually no shell at all. This effectively removes the user's shell access, ensuring they can only use their access for SFTP file transfers.

## Step 9: Jailing FTPS users

In order to jail our users, which we'll accomplish using [chroot](http://manpages.ubuntu.com/manpages/focal/en/man2/chroot.2.html){:target="_blank"}, we need to set some very specific permissions, and make a few changes to our configuration files.

First, make sure the owner of our FTPS parent directory, and all user subdirectories, matches our `guest_username`. The username is defined in our `vsftpd.conf` config, and in this case, it's `ftp`:

{% highlight bash %}
sudo chown -R ftp:ftp /home/vftp
{% endhighlight %}

Next, remove all group permissions on our parent FTPS directory:

{% highlight bash %}
sudo chmod 0555 /home/vftp
{% endhighlight %}

## Step 10: Jailing SFTP users

In order to jail our SFTP users, we'll need to create a group, under which all SFTP users must belong — we'll call it `sftponly`:

{% highlight bash %}
sudo addgroup sftponly
{% endhighlight %}

Now let's add our users to this group:

{% highlight bash %}
sudo usermod -a -G sftponly batman
sudo usermod -a -G sftponly robin
{% endhighlight %}

Next, we need to make some configuration changes to our SSH service. Open the [sshd_config](http://manpages.ubuntu.com/manpages/focal/en/man5/sshd_config.5.html){:target="_blank"} file:

{% highlight bash %}
sudo nano /etc/ssh/sshd_config
{% endhighlight %}

find this line:

{% highlight bash %}
Subsystem   sftp    /usr/lib/openssh/sftp-server
{% endhighlight %}

and replace with:

{% highlight bash %}
Subsystem   sftp    internal-sftp
{% endhighlight %}

What we're doing here, is defining the [external subsystem](http://manpages.ubuntu.com/manpages/focal/en/man5/sshd_config.5.html){:target="_blank"} (eg: file transfer daemon), which is started automatically after SSH login from the client. The `internal-sftp` value implements an in-process SFTP server that requires no support files when defining a `ChrootDirectory`. Basically, it simplifies the process allowing us to force a different filesystem root on our users (jail them).

Now lets create a conditional block, using Match, that will apply some options to any user belonging to the `sftponly` group:

{% highlight bash %}
# lock all users that are part of the
# `sftponly` group to our ChrootDirectory
Match Group sftponly
        ForceCommand internal-sftp -d /%u
        PasswordAuthentication yes
        ChrootDirectory /home/sftp
        PermitTunnel no
        AllowAgentForwarding no
        AllowTcpForwarding no
        X11Forwarding no
{% endhighlight %}

There's a few important things to understand about these options.

1. [ForceCommand](http://manpages.ubuntu.com/manpages/focal/en/man5/sshd_config.5.html){:target="_blank"} forces the execution of the command specified, ignoring any commands supplied by the client. By default, when an SSH user logs in, they would land in our `/home/sftp` directory, thereby allowing them to see all other users that might exist. While they won't be able to access those directories, it's better they don't see them at all. To fix this, we're going to force a directory change upon login, by passing the `-d <path>` argument, where `/%u` is our path relative to our ChrootDirectory, and `%u` is a token that represents the username. So on login, the user should automatically be brought to `/home/sftp/batman` without knowing it.

2. [PasswordAuthentication](http://manpages.ubuntu.com/manpages/focal/en/man5/sshd_config.5.html){:target="_blank"} you can set this to either `no` or `yes` depending if you want to allow SFTP users to authenticate with passwords. The alternative being private/public SSH keys, which are safer, though requires more involvement to manage. I recommend allowing passwords, so long as you are creating users with cryptographically strong passwords.

3. [ChrootDirectory](http://manpages.ubuntu.com/manpages/focal/en/man5/sshd_config.5.html){:target="_blank"} specifies the pathname of the directory to [chroot](http://manpages.ubuntu.com/manpages/focal/en/man2/chroot.2.html){:target="_blank"} to after the user authenticates. A requirement of Chroot, is that `root` be the owner of the jailed directory.

Lastly, lets set some required permissions — note that `0711` grants public execute, but limits read and write to the owner (root), as required by Chroot:

{% highlight bash %}
sudo chown root:root /home/sftp
sudo chmod 0711 /home/sftp
{% endhighlight %}

## User command script

In order to easily manage your FTP users, we're going to create a command script to do the work for us. This script will enable you to:

- List all existing users — `./path/to/users list`
- Add a new user — `./path/to/users add <username> <password>`
- Edit an existing user — `./path/to/users edit <username> <password>`
- Delete a user — `./path/to/users del <username>`

This script is written in Perl, and is largely just a series of linux commands. I've included links to some helpful documentation:

- [Perl DB_File](https://perldoc.perl.org/DB_File){:target="_blank"}
- [PAM_userdb](https://www.man7.org/linux/man-pages/man8/pam_userdb.8.html){:target="_blank"}

Create a new file — you can include the `.pl` extension if you prefer:

{% highlight bash %}
sudo nano /etc/vsftps/users
{% endhighlight %}

To help facilitate future updates, i'll include the script as a gist:

- 🤖 [users.pl](https://gist.github.com/phuze/8f66a26767500398d271b42013672536){:target="_blank"} — Perl script for managing users in a Berkeley DB within a Linux environment.

<!--{% gist 8f66a26767500398d271b42013672536 %}-->

## Remote control over API

As a final consideration, you could further integrate your FTP control system, by building an API. This API would SSH into your FTP server, and execute commands using the command script. You can then build a graphical interface to manage your FTP users.

Here's a quick example written in PHP, which makes use of [phpseclib](https://github.com/phpseclib/phpseclib){:target="_blank"}:

{% highlight php %}
namespace MyApi\Models;

use phpseclib3\Net\SSH2;
use phpseclib3\Crypt\PublicKeyLoader;

class FtpModel
{
    public function __construct() {
        $this->key = PublicKeyLoader::load(file_get_contents('/path/to/private/key'));
        $this->ssh = new SSH2('my.domain.com', 22, 30);
        if (!$this->ssh->login('username', $this->key)) {
          throw new Exception('Unable to establish SSH connection.');
        }
    }

    public function addUser(string $username, string $password) {
        $this->ssh->exec("sudo /etc/vsftpd/users add {$username} {$password}");
    }
}
{% endhighlight %}

and in Python — using [Paramiko](https://github.com/paramiko/paramiko){:target="_blank"}:

{% highlight python %}
import paramiko

def connect():
    conn = paramiko.SSHClient()
    conn.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    conn.connect('<host>', username='<user>', password='<pass>', key_filename='</path/to/private/key>')
    return conn

def main():
    ssh = connect()
    stdin, stdout, stderr = ssh.exec_command('sudo /etc/vsftpd/users add {$username} {$password}')
    print stdout.readlines()
    ssh.close()

main()
{% endhighlight %}

