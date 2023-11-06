---
title: "Recording system events with auditd"
date: 2022-12-11T21:15:13+05:30
draft: false
# showtoc: false
tags: 
 - linux
 - security
 - audit

# series:
description: auditing linux systems with auditd
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

Audits are critical for system administrators to detect security violations and track security-relevant information on their systems.
Anyone concerned about the security, stability, and proper operation of their Linux servers should conduct an audit. 

## How to do auditing in linux

One simple way is to use the `history` command to observe the shell's history, but this has many limitations. One of them is that this command is only applicable to the current user. You can still get around this by reading the `.bash_history` file in each user's home directory (given you have permissions to do so). 

### Audit framework in kernel.

The Linux audit framework is a better option.
Because it operates at the kernel level, it has a lot of visibility over almost everything. The Linux kernel sends significant events to user-space (`auditd`) so that they can be recorded in a file. This file can then be analysed on the host system or sent to a remote location for storage and analysis. 


## User-space auditd

The majority of Linux distributions come with `auditd` preinstalled, which begins and stops with the system (as a systemd service file). Using below command, you may determine whether the kernel was built using the audit options. 


```{linenos=false}
grep -i audit /boot/config-`uname -r`
```

On my system, it gives me below output (indicating kernel was built with auditing feature)

```{linenos=false}
CONFIG_AUDIT_ARCH=y
CONFIG_AUDIT=y
CONFIG_AUDITSYSCALL=y
CONFIG_AUDIT_WATCH=y
CONFIG_AUDIT_TREE=y
CONFIG_NETFILTER_XT_TARGET_AUDIT=m
CONFIG_IMA_AUDIT=y
CONFIG_KVM_MMU_AUDIT=y
```

Second thing you would want to check if the kernel thread process responsible for sending data to user-space is running. Check that with the `ps` command.

```{linenos=false}
sudo ps -aux | grep -i kauditd
```

This gives me below output (indicating that the thread is running)

```{linenos=false}
root       103  0.0  0.0      0     0 ?        S    11:39   0:00 [kauditd]
```

Final thing is to check the user-space service responsible to get the data from `kauditd`. To obtain definitive indications on systemd systems, use the commands listed below. 

```
systemctl is-active auditd          ## Returns: active/inactive
systemctl is-enabled auditd         ## Returns: enabled/disabled
```

*(**Note**: Feel free to check the source code at [`kernel/audit.c`](https://elixir.bootlin.com/linux/latest/source/kernel/audit.c). [^kernel_audit_c])*

[^kernel_audit_c]: https://elixir.bootlin.com/linux/latest/source/kernel/audit.c

### Configuring auditd

`Auditd` decides what to log and what not to log using a set of rules. These rules can be found in the `/etc/audit/rules.d/` folder. Auditd reads files from this folder on startup and generates the `/etc/audit/audit.rules` file automatically. **(This file should not be edited by hand.)**

`auditd` comes with a configuration file too. This file helps in changing the behaviour of the userspace `auditd` daemon. Default file on my system looks like below.

```{linenos=false}
# sudo cat -n /etc/audit/auditd.conf

     1	#
     2	# This file controls the configuration of the audit daemon
     3	#
     4	
     5	local_events = yes
     6	write_logs = yes
     7	log_file = /var/log/audit/audit.log
     8	log_group = root
     9	log_format = RAW
    10	flush = INCREMENTAL_ASYNC
    11	freq = 50
    12	max_log_file = 8
    13	num_logs = 5
    14	priority_boost = 4
    15	disp_qos = lossy
    16	dispatcher = /sbin/audispd
    17	name_format = NONE
    18	##name = mydomain
    19	max_log_file_action = ROTATE
    20	space_left = 75
    21	space_left_action = SYSLOG
    22	verify_email = yes
    23	action_mail_acct = root
    24	admin_space_left = 50
    25	admin_space_left_action = SUSPEND
    26	disk_full_action = SUSPEND
    27	disk_error_action = SUSPEND
    28	use_libwrap = yes
    29	##tcp_listen_port = 60
    30	tcp_listen_queue = 5
    31	tcp_max_per_addr = 1
    32	##tcp_client_ports = 1024-65535
    33	tcp_client_max_idle = 0
    34	enable_krb5 = no
    35	krb5_principal = auditd
    36	##krb5_key_file = /etc/audit/audit.key
    37	distribute_network = no
```

Some of these options are easy to understand, like:

- **log_file** : Tells the location of the audit log file.
- **max_log_file** : Defines the size of the log file in MB. If the size is reached, **max_log_file_action** is triggered.
- **space_left** : Triggers the **space_left_action** when the limit is reached.
- To include additional information in audit logs you need to change `log format` from **RAW** to `ENRICHED`. 
- `FLUSH = INCREMENTAL_ASYNC` will write the logs async instead of writing them on every write. 

While some of them needs more detailed explaination. In any case, always refer the man pages --> [`man (5) auditd.conf`](https://www.man7.org/linux/man-pages/man5/auditd.conf.5.html) [^man_5_auditd]. There you will find all the possible options and their supporting values to tune auditd as per your requirements.

[^man_5_auditd]: https://www.man7.org/linux/man-pages/man5/auditd.conf.5.html

After making changes to `auditd.conf`, restart the service to pick up new changes from config. My `centos 7` machine did not allow me to manually restart the service using `systemctl` but it worked just fine with `service auditd restart`. If you figure out why this happens, please let me know!


### Inspecting audit logs

We can see where the `auditd` logs are stored from the config file above. So we can always look through the log files and use the good old `grep` command to find what we're looking for.


But that is not the intended method. The `audit` package includes a number of helper commands to assist the sysadmin/analyst in quickly determining information from logs. 


Below are all the binary executable files provided by the `audit` package.... 

```{linenos=false}
## COMMAND:   rpm -ql audit | grep bin

/sbin/audispd
/sbin/auditctl
/sbin/auditd
/sbin/augenrules
/sbin/aureport
/sbin/ausearch
/sbin/autrace
/usr/bin/aulast
/usr/bin/aulastlog
/usr/bin/ausyscall
/usr/bin/auvirt
```

Let's start with `ausearch` for now. This program parses the audit log files and gives the information based on passed keywords.

There are a lot of options for this tool.. I'll mention few which I use most often.

- **`-i`** -- Interpret the logs. Translates numeric value in names.
- If you want to get raw logs, use **`-r`**.
- use **`-x`** to search based on executable name.
- If you know the event ID then search with **`-a`**.
- To search with message type, use **`-m`**. You can get the message type list by passing nothing or a wrong message type with the argument/flag.
- Use **`-k`** to search for specific key in log. You can configure your own key in the logs config. These keys helps to corelate the logs with the rules.


If you just want to get a report of everything that was logged, you can use `aureport` program which gives you a proper summary in a tabular form. 

```{linenos=false}
## COMMAND:   sudo aureport

Summary Report
======================
Range of time in logs: 01/01/1970 00:00:00.000 - 12/10/2022 16:24:29.088
Selected time for report: 01/01/1970 00:00:00 - 12/10/2022 16:24:29.088
Number of changes in configuration: 2
Number of changes to accounts, groups, or roles: 0
Number of logins: 0
Number of failed logins: 0
Number of authentications: 0
Number of failed authentications: 0
Number of users: 3
Number of terminals: 4
Number of host names: 1
Number of executables: 3
Number of commands: 1
Number of files: 0
Number of AVC's: 0
Number of MAC events: 0
Number of failed syscalls: 0
Number of anomaly events: 0
Number of responses to anomaly events: 0
Number of crypto events: 0
Number of integrity events: 0
Number of virt events: 0
Number of keys: 0
Number of process IDs: 42
Number of events: 240
```

### Writing custom audit rules

`auditd` also allows us to write our own rules. These rules will be read and applied when the service is restarted... or if you invoke `augenrules --load`.

For auditing, there are only three types of rules that can be defined:

1. Watches on the file system (watches the changes related to filesystem or on a particular path)

2. syscalls (checks if a specific syscall was executed and with what context)

3. control rules (these are used to modify the kernel configuration of linux audit)


That's all. This was all we needed to know before we started writing our first simple rule. 


```{linenos=false}
-w /etc/
```

The above rule will watch for all kinds of changes in `/etc/` folder... that means any (r)ead, (w)rite, (e)xecute or (a)ttribute change operations will be logged.

Let's write the above rule in a new file: `/etc/audit/rules.d/myrules.rules`... And check if it is picked up by auditd already. (I know it will not be picked, but it won't hurt to check)

```{linenos=false}
# sudo auditctl -l

No rules
```

Now, let's restart the service and try that again.

```{linenos=false}
# service auditd restart
# sudo auditctl -l

-w /etc -p rwxa
```

`auditd` has now loaded the rule, as expected. But there's more to it than just what we put in the file. It makes no difference, however, because it is implicitly adding `-p rwxa` to indicate that all of these operations should be monitored. 

The files still contain what we added... but the kernel has fully expanded rules.

```{linenos=false}
# sudo cat /etc/audit/rules.d/myrules.rules

-w /etc/

```

```{linenos=false}
# sudo cat /etc/audit/audit.rules

## This file is automatically generated from /etc/audit/rules.d
-D
-b 8192
-f 1
-w /etc/
```

With this rule in the kernel, all the operations made to `/etc` path will be recorded. To make things easy, think of all the **watch** rules as just fancy wrappers for syscall rules. Above rule can be written as below, and will still work the same.

```{linenos=false}
-a exit,always  -F dir=/etc -F perm=rwxa
```

Remove the previous rule and add the above rule to the same file. Restart the service again for the changes to take effect.

```bash{linenos=false}
$ sudo cat /etc/audit/audit.rules

## This file is automatically generated from /etc/audit/rules.d
-D
-b 8192
-f 1
-a exit,always  -F dir=/etc -F perm=rwxa

```

Auto-generated event is what we wrote in the file. Let's take a look what it looks like from kernel point of view.

```{linenos=false}
$ sudo auditctl -l

-w /etc -p rwxa
```


Told you, its practically the same. Now let's understand all the options in the new rule we wrote (obviously for better clarity on how it is same).

```{linenos=false}
-a exit,always  -F dir=/etc -F perm=rwxa
```

- `-a` : append rule to end of the list
- `exit,always` : always log when exiting a syscall.
- `-F` : build a rule based on (F)ield values
- `dir=/etc`: full path of directory to watch; watches recursively to whole subtrees.
- `perm=rwxa`: permission changes/access to monitor.

According to `man 8 auditctl`, **if  a field rule is given and no syscall is specified, it will default to all syscalls.** That means the above rule will work for all of the syscalls.



So far so good. Now what about control rules?? ..Or let's say *configure* rules as they help in configuring the behaviour of auditd itself.

These rules help in configuring/controling the behaviour of auditd. Read the man page for better and complete explanation. But I'll walk you through the ones we have already seen.... in the `/etc/audit/audit.rules` file.


```
-D
-b 8192
-f 1
```

- `-D` deletes all the previous rules from kernel rules list. This should be always on the top If you want to give someone a hard time, just put that in the end.(**please don't do it on production machines, it won't be funny**)

- `-b` sets the size for audit buffer. If you don't know what you are doing, leave it to the default.

- `-f` sets the failure mode that let's the kernel decide how to handle failures and critical errors. 0 is silent. Default is 1 (printk). Super secured environment should be using 2 (panic). 


### Pre-packaged audit rules


Most of the times, we don't really need to write our own audit rules, we can just use what other people have already worked upon. You can always find them with the help of your favorite search engine...but there are few already pre-packaged with `audit` and are already on your system (if you have installed the package)


```{linenos=false}
$ rpm -ql audit | grep '/usr/share/.*\.rules$'

/usr/share/doc/audit-2.8.5/rules/10-base-config.rules
/usr/share/doc/audit-2.8.5/rules/10-no-audit.rules
/usr/share/doc/audit-2.8.5/rules/11-loginuid.rules
/usr/share/doc/audit-2.8.5/rules/12-cont-fail.rules
/usr/share/doc/audit-2.8.5/rules/12-ignore-error.rules
/usr/share/doc/audit-2.8.5/rules/20-dont-audit.rules
/usr/share/doc/audit-2.8.5/rules/21-no32bit.rules
/usr/share/doc/audit-2.8.5/rules/22-ignore-chrony.rules
/usr/share/doc/audit-2.8.5/rules/23-ignore-filesystems.rules
/usr/share/doc/audit-2.8.5/rules/30-nispom.rules
/usr/share/doc/audit-2.8.5/rules/30-ospp-v42.rules
/usr/share/doc/audit-2.8.5/rules/30-pci-dss-v31.rules
/usr/share/doc/audit-2.8.5/rules/30-stig.rules
/usr/share/doc/audit-2.8.5/rules/31-privileged.rules
/usr/share/doc/audit-2.8.5/rules/32-power-abuse.rules
/usr/share/doc/audit-2.8.5/rules/40-local.rules
/usr/share/doc/audit-2.8.5/rules/41-containers.rules
/usr/share/doc/audit-2.8.5/rules/42-injection.rules
/usr/share/doc/audit-2.8.5/rules/43-module-load.rules
/usr/share/doc/audit-2.8.5/rules/70-einval.rules
/usr/share/doc/audit-2.8.5/rules/71-networking.rules
/usr/share/doc/audit-2.8.5/rules/99-finalize.rules
```
*( **NOTE:** The numbers in the filenames play a very important role. For auditd, the first rule found wins. So if there are 2 contradictory rules, the first one found will be applied and the second one will have no effect.)*

You can copy these rules, or just the ones you want to monitor, to `/etc/audit/rules.d/` folder and restart the service to pick up the new rules. Or you can use `augenrules --load` to load them without restarting the service.

### Hardening the audit

First step to harden the audit will be to ensude auditd's configuration is immutable. This can be done with `-e 2` control rule. Enabling this will prevent further changes in auditd's configurations. This being said, it is very obvious that this should be the last rule in the list.

Next step would be to store the logs into a centralized secure location. `Auditd` comes with a dispatcher program (`auditspd`) that can work with `auditsp-remote` plugin. This program too comes with it's own configuration file, which can be found at `/etc/audisp/audisp-remote.conf`.

*This package was not already installed on my system so I installed it with `sudo yum install -y audispd-plugins`. Once this is installed, `auditsp-remote.conf` will be there witing for you to edit.*

There are a few configuration changes you'll need to make to ensure that logs are sent to the remote server. The overall concept is to collect logs using auditd, then use a plugin to send logs to a central server while also disabling local logging of the same logs. This way, we won't have logs on the local system (saving disk space), and we can aggregate logs from multiple servers for analysis. 

Let's start it with one change at a time. First one will be to enable the remote logging plugin. To do that, we can make changes to `/etc/audisp/plugins.d/au-remote.conf`.

```{linenos=false}
## CHANGE active status to yes

active=yes
```

Then let our audit dispatcher know about the remote server where we want to dispatch the logs. This change will be made to `/etc/audisp/audisp-remote.conf`

```{linenos=false}
## Remote server name/IP and the port
remote_server = 192.168.56.10
port = 60
```

*As a dirty trick, I've started netcat on port 60 to listen to the incoming data from the host.*

Last thing is to disable the local logging for `auditd`. For that, make changes to `/etc/audit/auditd.conf`.
 
```{linenos=false}
## CHANGE write logs to no
write_logs = no
```

With this done, you have everything configured and ready to test. Now restart the **auditd** service and you'll start getting logs in netcat screen on remote system.


## Wrap-up

In this article, you learnt about how to do better auditing of your linux environment, with the help of `auditd`. You also learnt about how to write your own rules or get pre-packaged rules to generate specific audit logs... and ways to get required reports with the help of `ausearch` and `aureport` programs.

This article is not intended to be a complete guide for auditing. It's whole purpose is to get you started with the idea of auditing and using the `audit` package utilities. 

If you want to learn more about it, I suggest you to play around and read [RedHat's documentation on system auditing](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-system_auditing) [^rh_system_auditing]. And if you are stuck, use your favorite search engine or... RTFM.

[^rh_system_auditing]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-system_auditing

![](https://media.giphy.com/media/8dYmJ6Buo3lYY/giphy.gif#center)





