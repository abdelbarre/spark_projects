# How to install a FTPS server on Centos V7

This document describes the way on how to install and configure a FTPS service on Centos 7 by creating **virtual accounts** and by using the well-known **vsftpd** package.


## Disable the firewall
In the case whene you firewall is running, perform the following commands :

```shell
systemctl status firewalld   ## To Get the actual status of  your firewall
sudo systemctl stop firewalld  ## Stop the firewall service.
sudo systemctl disable firewalld  ##  Disable it 
```

## Install the Linux packages

First of all, install the following Linux packages:

```shell
sudo yum update
sudo yum install ftp ## 
sudo yum install lftp   ## LFTP is a sophisticated file transfer program supporting a number of network protocols (ftp, http, sftp, fish, torrent)
sudo yum install db4-utils # Hint : Use 'db-util' if the package 'db4-utils' doesn't exist 
```
## Install the vsftpd
Then let's talk about  **vsftpd**:

**vsftpd** is a GPL licensed FTP server for UNIX systems, including Linux. It is secure and extremely fast. It is stable. Don't take my word for it, though. Below, we will see evidence supporting all three assertions. We will also see a list of a few important sites which are happily using vsftpd. This demonstrates vsftpd is a mature and trusted solution.

**Features** 

Despite being small for purposes of speed and security, many more complicated FTP setups are achievable with vsftpd! By no means an exclusive list, vsftpd will handle:

    - Virtual IP configurations
    - Virtual users
    - Standalone or inetd operation
    - Powerful per-user configurability
    - Bandwidth throttling
    - Per-source-IP configurability
    - Per-source-IP limits
    - IPv6
    - Encryption support through SSL integration
    - etc...
    
For more information please check this [vsftppage](https://security.appspot.com/vsftpd.html#about).

Lets make our hands dirty 😃, How to install it?

```shell
sudo yum install vsftpd
sudo mkdir -p /var/vsftpd
sudo chown -R ftp:ftp /var/vsftpd
```

## Create a vsftp account to store the user sites
here we create the vsftp account to store the users sites

Here are the commands to do that :

```shell
sudo adduser vsftp  # create user vsftp
sudo usermod -aG ftp vsftp ## add user vsftp to the group ftp
sudo mkdir -p /home/vsftp/sites  ## create the repo which contains all user sites 
sudo chown -R vsftp:ftp /home/vsftp ## give the vsftp:ftp as user:group owner 
sudo chmod -R 775 /home/vsftp  ## give it the right access mode (read/write/execute :-))
exit
```
then we will see how to create users and passwords to connect into ftp.

## Create the user-password list

Create a text file including users and passwords:
```shell
cd /etc/vsftpd
sudo vim passwd.txt  ## in this file we add a list of users and passwords you want
```

Example of file containing the user **foo** with its password 

```
toto
totoPasswd
```

**Note and Hint:**

don't use underscores in login names.


Then we Generate the user-password database in the way to hash password, and to do that:

```shell
cd /etc/vsftpd
sudo rm passwd.db  ## remove the default passwd.db 
sudo db_load -T -t hash -f passwd.txt passwd.db ## generate the new passwd.db by hashing the passwd.txt
sudo chmod 600 passwd.db ## give the right access mode
```
**Hint And Note :**

Move your text file **passwd.txt** in a safe place, or drop it as you want 😉.

To sum up till now we install the ftp and lftp, then we install and configure vsftp,

## Create a user site

For each user, make the following tasks (see the example bellow with the user **toto**):
```shell
sudo mkdir /home/vsftp/sites/toto 
sudo chown -R ftp:ftp /home/vsftp/sites/toto 
sudo chmod -R 775 /home/vsftp/sites/toto
```

## Edit the configuration file

For more information please check this [page](https://www.centos.org/forums/viewtopic.php?t=29016).

Edit a new configuration file:

```shell
cd /etc/vsftpd
sudo mv vsftpd.conf vsftpd.conf.back
sudo vim vsftpd.conf
```

Example of configuration file content (example with TCP ports **21** and **41000 to 41005**):
```
# ======================== #
# = Basic configuration == #
# ======================== #
listen=YES
listen_port=990
pasv_enable=YES
pasv_min_port=41000
pasv_max_port=41005
connect_from_port_20=NO
anonymous_enable=NO
local_enable=YES
virtual_use_local_privs=YES
write_enable=YES
secure_chroot_dir=/var/vsftpd
pam_service_name=vsftpd
guest_enable=YES
user_sub_token=$USER
local_root=/home/vsftp/sites/$USER
chroot_local_user=YES
hide_ids=YES
allow_writeable_chroot=YES
# Enable 2 log files, one for 'wu-ftpd' style (/var/log/xferlog) and one for vsftpd style (/var/log/vsftpd.log)
dual_log_enable=YES

# ======================
# = Upload permissions =
# ======================
local_umask=002
chmod_enable=YES
file_open_mode=0755
```

Make the task below just once:
```shell
sudo mkdir -p /var/vsftpd
```
Edit the following file by setting SELINUX to permissive:
```shell
sudo vim /etc/selinux/config
```

Example of content:
```
...
set SELINUX=permissive
...
```

## Configure the PAM service

Edit a new **vsftpd** service file (do it just once):

```shell
sudo mv /etc/pam.d/vsftpd /etc/pam.d/vsftpd.back
sudo vim /etc/pam.d/vsftpd
```

File content:
```
#%PAM-1.0
auth       required     pam_userdb.so db=/etc/vsftpd/passwd
account    required     pam_userdb.so db=/etc/vsftpd/passwd
session    required     pam_loginuid.so
```

Create the service:
```shell
sudo systemctl enable vsftpd
```

## Stop/Start/Restart the service

To stop the service:
```shell
sudo systemctl stop vsftpd
```

To start the service:
```shell
sudo systemctl start vsftpd
```

To restart the service:
```shell
sudo systemctl restart vsftpd
```

## 💻 Test the FTP connection in local mode (non secured)

By using the **ftp** client:
```shell
ftp localhost
lftp -u user1 localhost
```

By using the **lftp** client (with the user **toto**):
```shell
lftp -u foo localhost
```

## EXTRA 😎  : Configure the FTPS service

Let's create a TLS certificate:
```shell
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout vsftpd.pem -out vsftpd.pem
```

Example of questions during the creation of the certificate:
```
Country Name (2 letter code) [XX]:fr
State or Province Name (full name) []:Charente
Locality Name (eg, city) [Default City]:La Rochelle
Organization Name (eg, company) [Default Company Ltd]:AKKA
Organizational Unit Name (eg, section) []:Data Intelligence
Common Name (eg, your name or your server's hostname) []:akdatahub.com
Email Address []:Denis.Grandjean@akka.eu
```

Move the certificate:
```shell
sudo mv vsftpd.pem /etc/vsftpd/
```

Open the **vsftpd** configuration file:
```shell
sudo vim /etc/vsftpd/vsftpd.conf
```

Set the listen port to 990:
```shell
listen_port=990
```

Then add the following lines:
```
# =====================
# = SSL configuration =
# =====================
rsa_cert_file=/etc/vsftpd/vsftpd.pem
rsa_private_key_file=/etc/vsftpd/vsftpd.pem
ssl_enable=YES
allow_anon_ssl=NO
ssl_tlsv1=YES
ssl_sslv2=NO
ssl_sslv3=NO
implicit_ssl=YES
# If you want to access in SSL mode only locally, then set to YES the two following lines
force_local_data_ssl=NO
force_local_logins_ssl=NO
# Mandatory for 'ftplib' client library
require_ssl_reuse=NO
```


