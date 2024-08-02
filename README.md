# Securing-Ubuntu

There are a couple things  that need to be install and configured. The following is a list objectives.

SELinux will be enforcing security policies
IPtables will provide firewall functionality
FTP is not allowed, and all file transfers must be encrypted during transport (SSL / SSH file transfer)
{Option} SSH key based authentication
Disable root access via SSH
Securing GRUB

## Part 1: Securing Access
Installing SELinux & Tools
```
sudo apt-get install policycoreutils selinux-utils selinux-basics -y
```
```
sudo selinux-activate
```
```
sestatus
```
Adding a new user

Root access via SSH will be disabled, and a standard user account will be used for administering the host. Whenever root privileges are required, sudo will be used. In this example, a user called Fred will be added. Fred is the sysadmin.
```
useradd fred 
passwd fred 
visudo (this will open the sudoers file /etc/sudoers)
```
1) Look for the line root ALL=(ALL) ALL and add the new user account below it. This will allow the new user account to have sudo privileges.
```
fred ALL=(ALL) ALL
```
2) Save the file (:wq)

Disable Root SSH Access & Change Default Port

Edit /etc/ssh/sshd_config and edit the following:

Disable Root SSH Access & Change Default Port

Edit /etc/ssh/sshd_config and edit the following:
```
service ssh restart
```
Now logout, and log back in as the new user. Use sudo -i to login with root privileges if needed.

{Option} Configuring SSH Key Based Authentication
By this stage you should be logged in with your new user account (not root). PuTTYgen will be used to generate an RSA key pair and configure PuTTY to access the server using the private key.

{Option} Generate the public/private key-pair
Launch PuTTYgen make sure SSH-2 RSA is selected and 2048 bits.
Copy the public key in the text box, which starts with ssh-rsa AAAA and save it in a text file somewhere safe (For example, public.key).
Save the private key (For example, private.ppk) and don’t lose it!

{Option} Copy the public key to the server
In order for public/private key authentication to work, your server must have a copy of the public key in ~/.ssh/authorized_keys. ~/ denotes your home directory (/home/bob), and you may need to create the .ssh directory first. Just make sure both .ssh and the authorized_keys file is owned by the user being authenticated (not root). Also the permissions need to be set as follows, otherwise you will get an error: Server refused our key
```
chmod 700 .ssh # chmod 644 authorized_keys
```
NOTE: If you still get the error, check that this file isn’t owned by root.

1) Paste the contents of the public key in to the file ~/.ssh/authorized_keys. If this directory or file doesn’t exist, create it.

The authorized_keys file should look something like this:
```
ssh-rsa AAAAB3NzaC1yc2EAAAABJQAAAQEA5KigO6JFR8AtaCULlxdZ7lSo3OpkUY3I2K
YzGR+BuGjEcJCgSn6OLYTUw2ygFs4VhOwZ5Pmiq9T2RskiK6/gMa0VVoFM18xUS9Emwdq3pxWSD
SQ0pZzGut0mhBix+h7xpu40oIalLE7JJSWsjvzacKEgjq06lRQgpqWEAtV3E+Ks5tqyJ5/3PiEn
LCQOCma9OXp/YbtS37/Xi15iSeXzfgSf+BnOPB+yh72xwPZfIIx8KL8cVaK9MkYeKdBPMwM5R1a
I0Ek8T/idginalL4m9j+QoD7ajnOnm4NvOoD//K7uozEXcBZGUiMFX17aAcY44hWM462zJQ/Ml/
y2sco4iQ== rsa-key-20170906
```
Load the private key into PuTTY

Launch PuTTY and configure the hostname under the Session category.
Click on Connection > Data and type the Auto-login username
Click on Connection > SSH > Auth and Browse to the location of the private key.
Click on Session and save your configuration.

Now when you launch PuTTY it will present the private key, and this will match the corresponding public key on the server. Only you will hold a copy of the private key.


{If the above Options are configured} Disable SSH Password Authentication

This step is optional but strongly recommended. If you disable SSH password based authentication then you can only access SSH with the private key you configured in the last step. We still need passwords in order for sudo to work, but this step will only access certificate based authentication when you SSH to the host.

Edit /etc/ssh/sshd_config and set the following:

```
 PasswordAuthentication no
```
Core Packages Install
```
sudo apt  install -y wget ntp gcc net-tools iptables-services yum-cron sshfs epel-release
```
## Part 2: IP tables

One of the first measures is to disable root SSH access, change the SSH port to something other than the default (port 22), and set a maximum number of authentication attempts in order to minimize the likelihood of a brute force attack. Once that is  done then  configure IP tables.  By this point Iptables should be installed and running.

If you want to clear the current rules, use iptables -F to flush. When you’re done, remember to save and restart service
```
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP

# Block syn flood attack
iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP

# Block XMAS packets
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP

# SSH Rate limit new connections (drop if more than 3 attempts in 60 seconds) and allow only established SSH connections
iptables -A INPUT -i eth0 -p tcp --dport 9922 -m state --state NEW -m recent --set --name SSH
iptables -A INPUT -i eth0 -p tcp --dport 9922 -m state --state NEW -m recent --update --seconds 300 --hitcount 4 --rttl --name SSH -j DROP
iptables -A INPUT -i eth0 -p tcp --dport 9922 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 9922 -m state --state ESTABLISHED -j ACCEPT

# Allow DNS Queries
iptables -A INPUT -i eth0 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow NTP
iptables -A OUTPUT -o eth0 -p udp --dport 123 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -i eth0 -p udp --sport 123 -m state --state ESTABLISHED -j ACCEPT

# Allow ICMP
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
iptables -A OUTPUT -p icmp -m icmp --icmp-type 0 -j ACCEPT

# Allow Grafana
iptables -A INPUT -i eth0 -p tcp --sport 3000 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 3000 -m state --state NEW,ESTABLISHED -j ACCEPT

#Allow Cerbro
iptables -A INPUT -i eth0 -p tcp --sport 9090 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 9090 -m state --state NEW,ESTABLISHED -j ACCEPT

#Allow LDAP HTTP
iptables -A INPUT -i eth0 -p tcp --sport 389 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 389 -m state --state NEW,ESTABLISHED -j ACCEPT

# Allow SQL
sudo iptables -A INPUT -i eth0 -p tcp --dport 3306 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -o eth0 -p tcp --sport 3306 -m conntrack --ctstate ESTABLISHED -j ACCEPT


#Graylog Input & Output for services and  and log collector. Zabbix
iptables -A INPUT -i eth0 -p tcp -s 10.200.6.70/255.255.255.0 --match multiport --dport 9000,9200,9300,5044,51411,2055,51420,51430,51412,12201,27017,10051,10050 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --match multiport --sport  9000,9200,9300,5044,51411,2055,51420,51430,51412,12201,27017 -m conntrack --ctstate ESTABLISHED -j ACCEPT


# Web Server (HTTP/HTTPS)
iptables -A INPUT -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT

# Web Browsing
iptables -A INPUT -i eth0 -p tcp --sport 80 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -p tcp --sport 443 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

# Allow Inbound/Outbound to Localhost
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

#Allow SMTP outbound (E.g Sendmail)
iptables -A INPUT -i eth0 -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT

# Log all dropped packets
iptables -N LOGINPUT
iptables -N LOGOUTPUT
iptables -A INPUT -j LOGINPUT
iptables -A OUTPUT -j LOGOUTPUT
iptables -A LOGINPUT -m limit --limit 4/min -j LOG --log-prefix "DROP INPUT: " --log-level 4
iptables -A LOGOUTPUT -m limit --limit 4/min -j LOG --log-prefix "DROP OUTPUT: " --log-level 4

# Set policies to drop everything else
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

Save and then restart:
# iptables-save > /etc/sysconfig/iptables
# systemctl restart iptables
```
Check Iptables after  save/restarting service
```
iptables -L -n -v
```
If need be check line number for deleting rule

```
iptables -n -L -v --line-numbers
```
```
iptables -D INPUT 4
```
Ubuntu  has a slight different way of using IPTABLES

Saving Rules

Iptables rules are ephemeral, which means they need to be manually saved for them to persist after a reboot.

On Ubuntu, one way to save iptables rules is to use the iptables-persistent package. Install it with apt like this:
```
sudo apt install iptables-persistent
```
During the installation, you will be asked if you want to save your current firewall rules.

If you update your firewall rules and want to save the changes, run this command:
```
sudo netfilter-persistent save
```
### Installing Fail2Ban

Fail2ban works by dynamically adding IP addresses to the firewall that have failed a certain number of login attempts. It’s very easy to install and configure.

```
sudo apt install fail2ban
systemctl status fail2ban.service
systemctl enable fail2ban
```
Create a basic configuration:
```
vi /etc/fail2ban/jail.local
```
```
[DEFAULT]
# Set a 1 hour ban
bantime = 3600

# Override /etc/fail2ban/jail.d/00-firewalld.conf:
banaction = iptables-multiport

[sshd]
enabled = true
```
Execute the following
```
sudo systemctl enable fail2ban

sudo systemctl start fail2ban

sudo systemctl status fail2ban
```
## Part 3: SELinux

Setting Permissive Mode

1) First, install required packages for SELinux:

2) Next SELinux set to ‘permissive’ mode and reboot:

```
sudo apt-get install -y policycoreutils policycoreutils-python selinux-policy selinux-policy-targeted libselinux-utils setroubleshoot-server setools setools-console mcstrans
```
Before actually enforcing SELinux policies, it is important to test SELinux with ‘permissive’ mode first. SELinux will log activity to /var/log/audit/audit.log, starting with “SELinux is preventing”, which will be useful to troubleshoot any processes, files or directories that would otherwise be restricted.
```
 vi /etc/sysconfig/selinux
```
Change SELinux to ‘permissive’ and then reboot.

SELINUX=permissive (:wq)

```
 reboot
```
You can use seinfo -t to list all SELinux context types, but if you combine it with grep you can narrow the search. Give this a try, seinfo -t | grep httpd_sys. This will show you all of the context types starting with httpd_sys. This will become useful later on when troubleshooting SELinux.


### Step 1: Setting the Correct SELinux Context 

Example of the website directories:
```
# semanage fcontext -a -t httpd_sys_content_t "/data/.sitehome(/.*)?" 
# restorecon -Rv /data/.sitehome
```
semange is another useful command that can be used to read and configure settings on network ports, interfaces, SELinux modules, file context and booleans.
```
restorecon -Rv
```
will set this security context recursively (-R) and will output any changes to the type (-v).


Step 2: Allow SSH on non-default port

Another change is the default SSH port from 22 to 9922. Again, using semanage the new port can be specified.
```
semanage port -a -t ssh_port_t -p tcp 9922
```
### Step 3: Allow Apache to use Sendmail

This time the setsebool command will be used to set the boolean for httpd_can_sendmail to 1. This will allow Apache (httpd) to send email.
```
 setsebool ­P httpd_can_sendmail 1
```
### Step 4 (Optional): Allow Apache to Read & Write directories with httpd_sys_content file types

If Apache/httpd is install the httpd_unified Boolean is turned off by default  . This does strengthen security, as any files and directories that need to be writable by Apache will require the httpd_sys_rw_content_t context. However,  this can prevent users from uploading files or installing plug-ins. If you decide to enable this and allow read, write and execute, then use the following command:
```
 setsebool -P httpd_unified 1
```
Troubleshooting SELinux

First, check the status of SELinux and make sure it is set to permissive.
```
sestatus
```
Check that SELinux is enabled, the policy is ‘targeted’ and the mode is set to ‘permissive’:
```
SELinux status: enabled
Loaded policy name: targeted
Current mode: permissive
```
When troubleshooting SELinux it can be useful to clear the audit log and then rebooting. This will make it easier to read the logs and not get confused with older entries while you are troubleshooting in permissive mode.

1) Clear the audit.log and reboot
```
 > /var/log/audit/audit.log
```
```
reboot
```
2) Use sealert command to check for issues in audit.log
```
sealert -a /var/log/audit/audit.log
```
NOTE: You can run a summary of audit log reports using aureport -a. This will provide a summary of reports from the audit log (/var/log/audit/audit.log). You can check the logs for today using
```
aureport -a -ts
```
Rinse and repeat. You may need to do this a few times, implementing the fix, clearing the log, rebooting. Once you are satisfied that there are no more issues being reported by SELinux, you can now set it to ‘enabled’!

# vi /etc/sysconfig/selinux

Rinse and repeat. You may need to do this a few times, implementing the fix, clearing the log, rebooting. Once you are satisfied that there are no more issues being reported by SELinux, you can now set it to ‘enabled’!
```
vi /etc/sysconfig/selinux
```
Change SELinux to ‘enforced’ and then reboot.
```
SELINUX=enforced (:wq)
```
```
reboot
```
Securing The Kernel After boot.

### Securing The Kernel After boot.

```
$ grub-mkpasswd-pbkdf2
Enter password:   ********
Reenter password: ********
PBKDF2 hash of your password is grub.pbkdf2.sha512.10000.800E[..].79C[..]

```
Define GRUB user (greg in the following example) using generated hash and declare it as a superuser inside /etc/grub.d/40_custom configuration file.

```

#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
# define superusers
set superusers="greg"
#define users
password_pbkdf2 greg grub.pbkdf2.sha512.10000.800EF[..].7977C[..]
```
Double check configuration and ensure the superuser and  defined use is correct before updating grub.

Install the modified configuration and test it afterward.
```
$ sudo update-grub
```























