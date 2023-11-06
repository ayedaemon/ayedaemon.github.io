---
title: "Pluggable Authentication Modules - Linux"
date: 2022-12-27T23:25:23+05:30
draft: false
# showtoc: false
tags: [linux, security]
# series:
description: Linux-PAM is a system of libraries that handle the authentication tasks of applications (services) on the system.
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

## PAM - What and Why

Authenticating a user to a service used to be a time-consuming process. The application had to be aware of all possible authentication mechanisms and had to be rebuilt every time a new authentication method was introduced... As a result, there was a significant amount of code repetition. Naturally, it was disliked by everyone!!


As a result, the concept of a middle-ware application responsible for user authentication to a service arose. And, Pluggable Authentication Modules (PAM), a collection of modules that act as a barrier between a service on your system and the service's user, were created. 

Modules can include a variety of functions, such as disabling login for specific users/groups, limiting resources, audting, and so on. PAM is now supported by the vast majority of major unix flavours, including AIX, HP-US, FreeBSD, and nearly all Linux distributions.


The big advantage here is that security is no longer a concern for the application: if PAM says "it's OK", it's OK. That simplifies things for both the application developer and the system administrator. 
 

## Understanding PAM

According to `man (8) pam`,

> Linux-PAM is a system of libraries that handle the authentication tasks of applications (services) on the system. The library provides a stable general interface (Application Programming Interface - API) that privilege granting programs (such as login(1) and su(1)) defer to to perform standard authentication tasks.

These libraries are typically configurable via defined arguments or dedicated configuration files. Internal behavior of the Linux-pam library is trivial from the standpoint of a sysadmin. The key point is to define the relationship between applications and the PAM. 


The below diagram gives an idea of how PAM works.


```goat



                                                 ┌────────────┐              ┌──────────┐
                    	┌───────────┐               │            │              │          │
                    	│           │               │   APP 2    │              │  APP 3   │
                    	│   APP 1   │               │            │              │          │
                    	│           │               └─┬──────────┘              └┬─────────┘
                    	└─┬─────────┘                 │    ▲                     │   ▲
                      	│    ▲                      │    │                     │   │
                      	│    │                      │    │                     │   │
                      	│    │                      │    │                     │   │
                      	│    │                      │    │                     │   │
                      	│    │                      │    │                     │   │
                      	│    │                      │    │                     │   │
                      	▼    │                      ▼    │                     ▼   │
             ┌──────────────┴───────────────────────────┴─────────────────────────┴──────────┐
             │                                                                               │
             │                                                                               │
             │                                    PAM                                        │
             │                                                                               │
             ├─────────────┬───────────────┬────────────────┬────────────────────────────────┤
             │ pam_unix.so │  pam_ldap.so  │ pam_tty_audit  │ (and other libraries)          │
             └─────┬───────┴───────┬───────┴────────────────┴────────────────────────────────┘
                  	│               │
                  	│               │
                  	│               │
                  	▼               ▼
              ┌─────────────┐  ┌─────────────┐
              │             │  │             │
              │ /etc/passwd │  │             │
              │             │  │    LDAP     │
              │ /etc/shadow │  │             │
              └─────────────┘  └─────────────┘


```

Assume the user attempts to log into `APP 1`, which checks the PAM to see if the user is authenticated and authorised. If the query is successful, PAM returns the status code `PAM_SUCCESS`; otherwise, it returns one of the other relevant codes.
The complete list of return codes and their meanings can be found in their github repository, which can be found [here](https://github.com/linux-pam/linux-pam/blob/b872b6e68a60ae351ca4c7eea6dfe95cd8f8d130/libpam/include/security/_pam_types.h#L29). [^pam_return_codes]


[^pam_return_codes]: https://github.com/linux-pam/linux-pam/blob/b872b6e68a60ae351ca4c7eea6dfe95cd8f8d130/libpam/include/security/_pam_types.h#L29


## Configuring PAM

PAM's main feature is the module configuration it offers. PAM looks at these text configuration files to determine what security actions to take for an application, and the administrator can add or remove new rules at any time. PAM is also extensible, which means that if we want to add new features (such as 2FA/MFA), we only need to change a few files and `login` can now use them. 

In RedHat based systems, all of the pam config files can be easily located with the below command.

```{linenos=false}
$ rpm -ql pam | grep /etc

/etc/pam.d
/etc/pam.d/config-util
/etc/pam.d/fingerprint-auth
/etc/pam.d/other
/etc/pam.d/password-auth
/etc/pam.d/postlogin
/etc/pam.d/smartcard-auth
/etc/pam.d/system-auth
/etc/security
/etc/security/access.conf
/etc/security/chroot.conf
/etc/security/console.apps
/etc/security/console.handlers
/etc/security/console.perms
/etc/security/console.perms.d
/etc/security/group.conf
/etc/security/limits.conf
/etc/security/limits.d
/etc/security/limits.d/20-nproc.conf
/etc/security/namespace.conf
/etc/security/namespace.d
/etc/security/namespace.init
/etc/security/opasswd
/etc/security/pam_env.conf
/etc/security/sepermit.conf
/etc/security/time.conf
```

There are two main directories here: `/etc/pam.d` and `/etc/security`. Both of these directories play important roles in configuring PAM behaviour.


Each file in the `/etc/pam.d` folder contains rules that are read by PAM at runtime. If the user attempts to login via `ssh`, he must be authenticated. PAM checks rules from the `sshd` file in the `/etc/pam.d/` folder after **sshd** sends an authentication request to PAM. If the file is present, the file's rules are read and a proper response is returned to the application. If the file is missing, the default behaviour is to read the rules from `other` file in same directory and act on them. 

Let's take a look at `/etc/pam.d/` folder to get better picture of what's in it.


```{linenos=false}
$ ls -l /etc/pam.d

-rw-r--r--. 1 root root  192 Feb  2  2021 chfn
-rw-r--r--. 1 root root  192 Feb  2  2021 chsh
-rw-r--r--. 1 root root  232 Apr  1  2020 config-util
-rw-r--r--. 1 root root  287 Jan 13  2022 crond
lrwxrwxrwx. 1 root root   19 Nov 18 13:01 fingerprint-auth -> fingerprint-auth-ac
-rw-r--r--. 1 root root  702 Nov 18 13:01 fingerprint-auth-ac
-rw-r--r--. 1 root root  796 Feb  2  2021 login
-rw-r--r--. 1 root root  154 Apr  1  2020 other
-rw-r--r--. 1 root root  188 Apr  1  2020 passwd
lrwxrwxrwx. 1 root root   16 Nov 18 13:01 password-auth -> password-auth-ac
-rw-r--r--. 1 root root 1033 Nov 18 13:01 password-auth-ac
-rw-r--r--. 1 root root  155 Jan 25  2022 polkit-1
lrwxrwxrwx. 1 root root   12 Nov 18 13:01 postlogin -> postlogin-ac
-rw-r--r--. 1 root root  330 Nov 18 13:01 postlogin-ac
-rw-r--r--. 1 root root  681 Feb  2  2021 remote
-rw-r--r--. 1 root root  143 Feb  2  2021 runuser
-rw-r--r--. 1 root root  138 Feb  2  2021 runuser-l
lrwxrwxrwx. 1 root root   17 Nov 18 13:01 smartcard-auth -> smartcard-auth-ac
-rw-r--r--. 1 root root  752 Nov 18 13:01 smartcard-auth-ac
lrwxrwxrwx. 1 root root   25 Nov 18 12:57 smtp -> /etc/alternatives/mta-pam
-rw-r--r--. 1 root root   76 Apr  1  2020 smtp.postfix
-rw-r--r--. 1 root root  904 Nov 24  2021 sshd
-rw-r--r--. 1 root root  540 Feb  2  2021 su
-rw-r--r--. 1 root root  200 Oct 14  2021 sudo
-rw-r--r--. 1 root root  178 Oct 14  2021 sudo-i
-rw-r--r--. 1 root root  137 Feb  2  2021 su-l
lrwxrwxrwx. 1 root root   14 Nov 18 13:01 system-auth -> system-auth-ac
-rw-r--r--. 1 root root 1031 Nov 18 13:01 system-auth-ac
-rw-r--r--. 1 root root  129 Sep  1 14:57 systemd-user
-rw-r--r--. 1 root root   84 Nov 24  2021 vlock

```

There are more files than what the rpm command above revealed. The reason for this is straightforward: we examined files installed by the `pam` package itself. Other files are installed by the packages that they belong to. The `openssh-server` package, for example, installed the `sshd` file. 

```{linenos=false}
$ rpm -qf /etc/pam.d/sshd

openssh-server-7.4p1-22.el7_9.x86_64
```

We now know that pam has rule files for each application as well as a default `other` file for all applications that do not have dedicated rule files. We won't always need this, but it's a good idea to keep it in the back of our minds. 

Let's take a closer look at these rules from the `sshd` file. 


```{linenos=false}
$ cat /etc/pam.d/sshd


#%PAM-1.0
auth	   required	pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare

account    required     pam_nologin.so
account    include      password-auth

password   include      password-auth
# pam_selinux.so close should be the first session rule

session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```

Lines beginning with `#` are clearly identified as comments, while the rest of the lines contain a single rule in a line.

Each rule follows a similar structure but uses different keywords. The generic rule syntax looks like this: 

```{linenos=false}
type    control    module [modules arguments]
```

There are 4 types of `type` in the PAM rules file.

- **auth** : rules for authentication.
- **account** : rules for account management, like expired passwords and allowed time of login.
- **password** : rules for password management, like checking password quality. These rules are only used when applications are changing the password used for auth.
- **session** : rules for session management. They typically run at the start or end of the session.

And there are 6 common types of `control`:

- **required** : if it fails, everything fails; if it passes, go to next.
- **sufficient** : if it passes, everything passes; if it fails, go to next.
- **requisite** : same as required- but stops on error.
- **optional** : pam ignores it (pass or fail); if this is the only module in stack then it decides if fail or pass.
- **include** : include rules from other pam files. if stack fails, return control to application.
- **substack** : works like include. but if the substack fails, return to the parent stack instead of giving control back to application.


The `module` (and any parameters, if any) follows. By default, PAM will look for modules in the `/usr/lib64/security` directory, but you can prevent this behaviour by specifying the absolute path of the module. Some modules rely on external configuration files, which can be found in the `/etc/security` directory. 


### Few common modules

Before delving into the actual PAM rules files, we should first understand how a few of the most common modules behave. This will make interpreting the rules from the rules file much easier. 

- `pam_succeed_if.so`

This module is designed to succeed or fail auth based on the characterstics of the user trying to log in and the arguments passed to the module. If all the arguments passed to the module matches the characterstics of the user trying to log in, then and only then, this module returns success.


- `pam_selinux.so`

This command sets the apropriate selinux security context. This can be used to set context when a session starts and restore it back.


- `pam_permit.so`

This is the simplest of all. It just permits acess and does nothing else. With that said, you should consider this module very dangerous in wrong hands.


- `pam_limits.so`

As its name suggests, this module sets the limits on the system resources for a user-session. Root user(uid=0) are also affected with this module.

There are many limits that can be configured, so there is a dedicated folder to host configuration files for it - `/etc/security/limits.d`. Alternatively, there is a `/etc/security/limits.conf` file. But its a good practice to have separate config files if possible.

- `pam_pwquality.so`

This module was developed by RedHat. The only action of this module is to prompt the user for the password and check its strength. To check its strenght, the modules uses a dictionary (of weak password) to see it the entered password is part of it. If the password is not in the list, then that password is checked against a set of rules defined by admin. 

These rules are configurable either by the use of module arguments or `/etc/security/pwquality.conf` config file.


- `pam_rootok.so`

This rule authenticates the user if the real uid is 0. No questions asked! 

If you don't know about what is a real and effective UID, read this [^real_user_uid].

[^real_user_uid]: (https://stackoverflow.com/questions/32455684/difference-between-real-user-id-effective-user-id-and-saved-user-id)

- `pam_faildelay.so`

This module sets the delay on failure. Like when a user types wron password, it fails and the next prompt is delayed by this module. If the delay is not given, then it will use `FAIL_DELAY` from `/etc/login.defs`.


- `pam_unix.so`

This is the standard unix authnetication module. Usually it uses `/etc/passwd` and `/etc/shadow` (if shadow is enabled) to authenticate the user. There are many tasks that can be performed by this module like checking the expire or last change of the password. 

The session component of this module logs when user logins or logs out of the system.

- `pam_deny.so`

Just like `pam_permit.so`, this module is very simple and straightforward. This denies the access to everybody.


- `pam_warn.so`

This module logs the service, terminal, user, or anything to syslog. This module always return `PAM_IGNORE`, so it just log events and have no participation in authentication process apart from that.


We're almost there now. Before we go any further, there are a few more things we should consider:


- PAM rules are parsed from top to bottom. If a *sufficient* rule is passed, then none of the below rules will be checked.

- Some of the rules start with `-` character, indicating that PAM should ignore them silently if the module is missing.

- If you want to know anything about a module, there is usually a man page available. The majority of the manpages for these modules provide examples of usage and return types.


- Making changes to PAM files has immediate effect. You do not need to restart. As a result, it's a good idea to keep a backup of the files before making any changes. 

- Any error in the PAM files has the potential to log you out of your system permanently. Keeping a live root shell while testing is therefore beneficial. If you made a mistake, you can undo your changes using this shell. 


## Usecases

### enforcing strong passwords

Let's apply everything we've learned so far to observe how PAM responds to password changes made with the `passwd` utility.

We now know that the `/etc/pam.d/passwd` file will be used by passwd (if it exists; else, `/etc/pam.d/other` will be read). This file will provide the procedures to be followed when sshd is used for any type of [authentication and authorization](https://www.cloudflare.com/learning/access-management/authn-vs-authz/). 

This makes it obvious for us to go and checkout the `/etc/pam.d/passwd` file... 

```{linenos=false}
$ cat /etc/pam.d/passwd

#%PAM-1.0
auth       include	system-auth
account    include	system-auth
password   substack	system-auth
-password   optional	pam_gnome_keyring.so use_authtok
password   substack	postlogin
```

Analysing this file let us know that this module **includes** `system-auth` rules file. So we'll now inspect the rules mentioned in `/etc/pam.d/system-auth` file.

```{linenos=false}
$ cat /etc/pam.d/system-auth

#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        required      pam_faildelay.so delay=2000000
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
auth        required      pam_deny.so

account     required      pam_unix.so
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
account     required      pam_permit.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so

```

Only the `password` **type** is relevant for our intended task out of all of these rules. 


```{linenos=false}
password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    required      pam_deny.so
```


- `requisite	pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=`

This rule checks the quality of password using some predefined configuration that are mentioned in `/etc/security/pwquality.conf` or can be explicitely dictated via module arguments. The `try_first_pass` option tells to load the password from previous rule (if any), else this module will make prompt user for password. `local_users_only` option will tell `pam_pwquality.so` module to ignore the users that are not in the `/etc/passwd` file. `retry` option is the number of tries a user gets to pick an acceptable password before the module returns an error. By default, the prompt the user gets when entering their password is "New password:". If the administrator sets `authtok_type=FOO`, the prompt becomes "New FOO password:". Here the default behaviour will be expected.


- `sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok`

For pam_unix, the sha512 option means use a password hashing routine based on the SHA512 algorithm. blowfish is also supported along with several other, less secure, choices. The shadow option means maintain password hashes in a separate /etc/shadow file that is only readable by the root user. This option should always be set. nullok means allow user accounts that have null password entries. Personally, I would recommend removing this option.


- `required pam_deny.so`

If the above modules failed, this should return with a deny message.


Now, say you want to enforce the following policy... 


    - prompt 2 times for password in case of an error (retry option)
    - 12 characters minimum length
    - at least 6 characters should be different from old password when entering a new one (difok option)
    - at least 1 digit (dcredit option)
    - at least 1 uppercase (ucredit option)
    - at least 1 lowercase (lcredit option)
    - at least 1 other character (ocredit option)
    - cannot contain the words "qwerty" and "password"
    - enforce the policy for root as well.


... necessary changes that are needed to make are as below


```{linenos=false}
password    requisite     pam_pwquality.so try_first_pass local_users_only retry=2 minlen=12 difok=6 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1 [badwords=qwerty password] enforce_for_root
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    required      pam_deny.so
```

The changes to the `system-auth` file described above will affect all applications that rely on that rule file. If changes are made to the `passwd` file, they will only affect the `passwd` utility.


We can also use the `authconfig` utility to make nearly all of these changes without having to interact directly with the associated files.
More information can be found [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system-level_authentication_guide/authconfig-pwd#authconfig-pwd-cmd) . [^redhat_authconfig]

[^redhat_authconfig]: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system-level_authentication_guide/authconfig-pwd#authconfig-pwd-cmd



### lock out at multiple failed attempts

We can also use PAM to configure the system so that if a single user makes multiple failed attempts, the PAM will lock out that user for a set period of time. Similarly to how your mobile device locks out the user for the next few hours if multiple login attempts fail.


This is possible with the `pam faillock` module. Because there is no dedicated config file for this module, all configuration will be done through module arguments. You can either manually edit the rule files with your favourite text editor or use the authconfig utility to make changes.

Before you begin, you must determine whether or not `pam_faillock` is enabled.
This can be verified using 


```{linenos=false}
authconfig --test | grep pam_faillock
```

For me the default output was

```{linenos=false}
pam_faillock is disabled (deny=4 unlock_time=1200)
```

So I had to enable it via `authconfig --enablefaillock` and along with that you can pass module arguments via  `authconfig --faillockargs=<module_options>` flag.

Or, one can combine both of the actions in a single command like

```{linenos=false}
authconfig --enablefaillock --faillockargs="fail_interval=30 deny=3 unlock_time=3600" --update
```

The above command will add `pam_faillock` rule in all of the relevant rules file (located in `/etc/pam.d/` directory). And the rest of the commands will configure the behaviour of the `pam_faillock` module. All of the module options can be checked with `man pam_faillock`.

Now the output of below command is slight different.

```{linenos=false}
authconfig --test | grep pam_faillock
```

Output:

```{linenos=false}
pam_faillock is enabled (fail_interval=30 deny=3 unlock_time=3600)
```

You can list the failed login attempts with the `faillock` command.


### TTy auditing ( \*cough*  keylogging  \*cough*)


Audit system (In Redhat or similar linux distros) uses `pam_tty_audit` PAM module for auditing of TTY input. When user logins, this module logs all keystrokes that user makes to `/var/log/audit/audit.log` file.

Since this depends on `auditd` service and requires that to be configured and running properly.

There is no `authconfig` flag that can enable this (atleast, there is none in centos 7.5), so we'll have to follow the traditional way of editing files manually to configure it.

This module only provides support for **session** type. That means we can only add **session** rules to our required files and it'll take effect immidiately for that service.

My idea is to add this rule in `/etc/pam.d/system-auth` and `/etc/pam.d/password-auth`.

```{linenos=false}
session     required    pam_tty_audit.so  enable=*
```


The above rule will capture all of the tty inputs as it is and store them in `/var/log/audit/audit.log` file (by default). You can easily `grep` stuff from that or use a proper tool to query the logs... like `aureport`.

`aureport --tty` command filters all `TYPE=tty` logs events from the file and display them in very human readable format.

### backdooring

Till this point, we have learnt a lot about PAM and it is time to rethink on the basics once again. PAM has 3 components: **user**, **password** and **service**. It's role is to authenticate a *user* to a *service* with provided *password*.

It works elegantly with the help of some service files of same name as of the service itself. These files are located in `/etc/pam.d/` directory. Each file has one or more rules that helps PAM to take proper decisions.

There are a lot of methods with which we can backdoor the PAM system. One of the many ways is to use `pam_exec.so` module to run an arbitrary command at each PAM based event. Another way could be to add rule using `pam_permit.so` module, that will skip the required checks to authenticate user for that service.

The above techniques are very noisy and are easily detected. More sophesticated attacks would include replacing the original module with custom compiled infected module on target system... or function hooking via `LD_PRELOAD` technique.

I'll leave the practical part upto you. Please don't do anything stupid or unethical on production server or any other system without the owner's consent. 

Keep it healthy and stay safe!!
 

## Resources:

- https://likegeeks.com/linux-pam-easy-guide/
- https://aplawrence.com/Basics/understandingpam.html
- https://www.linux.com/news/understanding-pam/
- https://developer.ibm.com/tutorials/l-pam/
- https://github.com/linux-pam/linux-pam (Source code)
- https://wiki.archlinux.org/title/PAM#Examples
- https://lwn.net/Articles/470764/ (A look at PAM face-recognition authentication)
- https://lwn.net/Articles/523199/ (Google Authenticator for multi-factor authentication)
