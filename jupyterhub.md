---
layout: default
title: JupyterHub on CentOS 7
---

JupyterHub on CentOS 7
======================

CentOS 7 + Active Directory + SELinux + Anaconda + JupyterHub + SudoSpawner

The purpose of this document is to demonstrate the steps necessary to run 
JupyterHub with sudospawner as a limited user using Anaconda on CentOS 7 
with SELinux enabled. This isn't intended to address any particular security 
considerations. 

<!--
ANACONDA_ROOT=/opt/anaconda3
JUPYTERHUB_USER=jupyterhub
JUPYTERHUB_CONF_DIR=/etc/jupyterhub
JUPYTERHUB_RUN_DIR=/srv/jupyterhub
JUPYTERHUB_PORT=8443
JUPYTERHUB_USER_GROUP=jupyterhub_users
-->


Active Directory
----------------

Setup Active Directory authentication (not covered here). I'm using `realm` 
to join the domain with the option `--client-software=winbind`. In 
`/etc/samba/smb.conf`, I set `template homedir = /home/%D/%U` and 
`winbind use default domain = yes`, do `rid`-based id mapping, and comment out 
all shares. My setup uses `rid` mapping to create a consistent, predictable,
deterministic relationship between AD users and linux uids; this is not
necessary for a stand-alone system.


Firewall Rules
--------------

Use the firewall to forward connections on the standard (privileged) https 
port (443) to the (unprivileged) port used by jupyterhub (8443). Replace
`web_zone` with whatever zone your interface uses.

```
firewall-cmd --permanent --zone=web_zone --add-service=https
firewall-cmd --permanent --zone=web_zone --add-forward-port=port=443:proto=tcp:toport=8443
firewall-cmd --reload
```

Install Everything
------------------

```
yum install wget bzip2 policycoreutils-python git epel-release
yum install nodejs npm
npm install -g configurable-http-proxy
wget http://repo.continuum.io/archive/Anaconda3-2.5.0_Linux-x86_64.sh
bash Anaconda3-2.5.0_Linux-x86_64.sh -b -p /opt/anaconda3
export PATH=/opt/anaconda3/bin:$PATH
pip install jupyterhub
pip install notebook
pip install git+https://github.com/jupyter/sudospawner
```


PAM Adjustments
---------------

#### Create home directories on sudo

Add additional session line to `/etc/pam.d/sudo`:

```
session    optional    pam_oddjob_mkhomedir.so umask=0077
```

#### Limit authentication to the jupyterhub users group

Edit `/etc/pam.d/system-auth` and `/etc/pam.d/password-auth` adding `require_membership_of=jupyterhub_users` to the end of the lines starting with `auth    sufficient    pam_winbind.so`.


Create limited user
-------------------

```
useradd --system --user-group --home-dir /srv/jupyterhub --shell /sbin/nologin jupyterhub
```


Set sudo permissions
--------------------

Create file `/etc/sudoers.d/jupyterhub` with the following contents:

```
# Using group instead of explicit list of users

# the command(s) the Hub can run on behalf of users without needing a password
# the exact path may differ, depending on how sudospawner was installed
Cmnd_Alias JUPYTER_CMD = /opt/anaconda3/bin/sudospawner

# actually give the Hub user permission to run the above command on behalf
# of the above users without prompting for a password
jupyterhub ALL=(%jupyterhub_users) NOPASSWD:JUPYTER_CMD

Defaults:jupyterhub exempt_group=jupyterhub
Defaults:jupyterhub !requiretty
```

Set permissions on the file:

```
chmod 0440 /etc/sudoers.d/jupyterhub
```



SELinux Policy
--------------

Create file `jupyterhub.te` with the following contents:

```
module jupyterhub 1.0;

require {
	type unconfined_t;
	type oddjob_mkhomedir_exec_t;
	## The below line shouldn't be needed if only authenticating AD users.
	#type chkpwd_exec_t;
	type sudo_exec_t;
	type unconfined_service_t;
	type oddjob_t;
	class process transition;
	class file entrypoint;
}

#============= oddjob_t ==============
allow oddjob_t unconfined_service_t:process transition;

#============= unconfined_service_t ==============
allow unconfined_service_t oddjob_mkhomedir_exec_t:file entrypoint;
allow unconfined_service_t unconfined_t:process transition;

#============= unconfined_t ==============
## The below line shouldn't be needed if only authenticating AD users.
#allow unconfined_t chkpwd_exec_t:file entrypoint;
allow unconfined_t sudo_exec_t:file entrypoint;	
```

Compile, package, and install the module:

```
checkmodule -M -m -o jupyterhub.mod jupyterhub.te
semodule_package -o jupyterhub.pp -m jupyterhub.mod
semodule -i jupyterhub.pp
```

Self-Signed SSL Cert
--------------------

```
openssl req -batch -nodes -new -x509 -days 365 -keyout server.key -out server.crt
```


Setup run directory
-------------------

```
mkdir /srv/jupyterhub
openssl rand -hex 1024 > /srv/jupyterhub/cookie_secret
cp server.crt /srv/jupyterhub
cp server.key /srv/jupyterhub
cp cookie_secret /srv/jupyterhub
chown -R jupyterhub:jupyterhub /srv/jupyterhub
chmod -R o-rwx /srv/jupyterhub
chmod 600 /srv/jupyterhub/server.key
```

	
JuyterHub config
----------------

```
mkdir /etc/jupyterhub
cd /etc/jupyterhub
jupyterhub --generate-config
```
	
Edit settings in `jupyterhub_config.py`:
	
```
c.JupyterHub.spawner_class = 'sudospawner.SudoSpawner'
c.JupyterHub.cookie_secret_file = '/srv/jupyterhub/cookie_secret'
c.JupyterHub.port = 8443
c.JupyterHub.ssl_cert = '/srv/jupyterhub/server.crt'
c.JupyterHub.ssl_key = '/srv/jupyterhub/server.key'
```


Setup service
-------------

Create file `/etc/systemd/system/jupyterhub.service` with the following contents:

```	
[Unit]
Description=JupyterHub
After=syslog.target network.target

[Service]
Type=simple
User=jupyterhub
Group=jupyterhub
Environment="PATH=/opt/anaconda3/bin:/usr/local/bin:/bin:/usr/bin"
WorkingDirectory=/srv/jupyterhub
ExecStart=/opt/anaconda3/bin/jupyterhub -f /etc/jupyterhub/jupyterhub_config.py
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```
systemctl daemon-reload
systemctl enable jupyterhub
systemctl start jupyterhub
```



