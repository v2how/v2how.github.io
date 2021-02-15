---
title: How to Build a V2Ray Tor On-Ramp on Ubuntu 20.04
date: 2021-02-15
tags: ["tor", "v2ray", "ubuntu"]
---

### Introduction

Are you in a country where the [Tor Project website](https://www.torproject.org) is blocked, the Tor protocol is blocked, and Tor bridges are also blocked? 

If so, this post offers a last resort for situations when all other methods of reaching Tor have failed. You'll learn how to create a Tor on-ramp on an Ubuntu 20.04 server located outside your country. With this method you can bypass censorship and connect to the Tor network from your existing browser.

By following along, you'll see how to create an Nginx camouflage web server and a hidden V2Ray server. The V2Ray server will relay your traffic to and from the Tor network. As a final step, you'll see how to test your Tor on-ramp using the Qv2ray graphical user interface.

At the end of this tutorial, you'll have a working and tested Tor on-ramp.

## Prerequisites

Before you begin this tutorial, you will need:

- A virtual private server (VPS) running Ubuntu 20.04. You can rent servers from providers such as [Google Cloud](https://cloud.google.com) or [Vultr](https://www.vultr.com). This server needs to be located in a country that does not block Tor.
- Your own domain name, which may be [free](https://www.freenom.com) or [paid](https://www.namesilo.com).
- A DNS type `A` record pointing from the fully qualified domain name of your server to the server's IP address. Usually you set up the DNS record at your domain name registrar. 

## Step 1 — Logging In as Root

SSH into your server. On Linux and macOS, you can use the terminal command `ssh` to reach your server. On Windows, you can either use [PowerShell](https://docs.microsoft.com/en-us/powershell) or a graphical user interface (GUI) such as [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty) or [XSHELL](https://www.netsarang.com).

If you're not logged in as `root`, then become `root` as follows. If you don't already know the root password, then set it by issuing the command:

```bash
sudo passwd root
```

The `sudo` command allows non-root users to issue root commands. The `passwd` command allows you to set the password for yourself or another user, which in this case is the user named `root`.

You are prompted to enter your own password:

```bash
[sudo] password for amy:
```

Enter your own (non-root) password.

You are then prompted to enter a new password for the user `root`:

```bash
New password:
```

Choose and enter the new password for `root`.

You are prompted to confirm the new password by retyping it:

```bash
Retype new password:
```

Enter the same new password for `root`.

If all is well, you will receive a feedback message:

```bash
passwd: password updated successfully
```

Now you know the root password, become `root`:

```bash
su -
```

The `su` command (short for switch user) allows you to switch to another user id, so that you can run commands with that user's privileges. Since no specific user is named, you will by default switch to the user `root`.  The hyphen (`-`) means create a new login environment for the new user, which will be completely separate from your existing login environment.

You are prompted to enter the password for `root` that you set a few moments ago:

```bash
Password:
```

Enter the root password.

You will notice that your command prompt changes from a dollar sign to a hash sign:

```bash
#
```

This is a reminder that your are logged in as `root`. Now that you are `root`, you do not need to prefix privileged commands with `sudo`.

Update your package lists, and upgrade all your packages to the latest version:

```bash
apt update && apt upgrade -y
```

You will see package upgrade messages in your terminal.

## Step 2 — Enabling the Firewall (Optional)

We recommend that you protect your server and especially port `22` (the SSH port) by installing a firewall. In this tutorial, you'll install and configure the uncomplicated firewall (`ufw`).

On many Ubuntu systems, `ufw` is already installed. Check that `ufw` is installed on your system:

```bash
apt list ufw
```

You should see a message:

```bash
ufw/focal,now 0.36-6 all [installed]
```

If you do not see the message stating that `ufw` is already installed, then install the software package now:

```bash
apt install ufw
```

Now you'll set up the firewall. If you can guarantee that you will always SSH into your server from a single IP address, then limit SSH access to just that single address. For example, if your IP address is `12.12.12.12`, then issue the command: 

```bash
ufw allow from 12.12.12.12/32 to any port 22 proto tcp
```

When you execute that command, you will see a message:

```bash
Rule added
```

If limiting access to a single IP address is not possible, but you can guarantee you will always connect from a known IP address range, then limit SSH access to just that range. For example, if you are in an office which always uses the IP address range `12.12.12.0/24`, then issue the command:

```bash
ufw allow from 12.12.12.0/24 to any port 22 proto tcp
```

If limiting access to an IP address range is not possible, then you will have to open the SSH port for access from the entire world:

```bash
ufw allow ssh
```

In any case, we recommend that you further protect port `22` by using SSH key authentication in preference to password authentication. SSH key pair generation and upload to your server are out of scope for this tutorial. You can find tutorials on this subject on the web.

Open ports `80` (the HTTP port) and `443` (the HTTPS port). These will be used for the Nginx camouflage web server:

```bash
ufw allow http
ufw allow https
```

Enable UFW:

```bash
ufw enable
```

You will see a warning message:

```bash
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```

Type `Y` and press `ENTER` to confirm that you wish to enable `ufw`. You will see a message:

```bash
Firewall is active and enabled on system startup
```

Check the status of your firewall:

```bash
ufw status
```

You will see a firewall status display that looks like the following:

```bash
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       12.12.12.0/24
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
80/tcp (v6)                ALLOW       Anywhere (v6)
443/tcp (v6)               ALLOW       Anywhere (v6)
```

If your VPS provider implements the concept of security groups, you will also need to open your security group assignments for ports `22`, `80`, and `443` in their control panel.

## Step 3 — Creating a Camouflage Website

Now you'll install Nginx and obtain an SSL certificate. The purpose of Nginx is to camouflage your V2Ray server.

Install the Nginx web server package:

```bash
apt install nginx -y
```

If Nginx was not already installed, you will see the package installation messages.

Edit the default virtual host configuration file, `/etc/nginx/sites-available/default`:

```bash
nano /etc/nginx/sites-available/default
```

Find the line containing `server_name _;`. Change it so that it refers to the fully qualified domain name of your server:

```bash
        server_name your_domain;
```

Write the file out to disk, and exit the editor.

Restart Nginx with your revised change configuration:

```bash
systemctl restart nginx
```

You will use `snap` to install `certbot`, the Let's Encrypt client. First install the `core` snap:

```bash
snap install core
```

After the snap is installed, you will see a confirmation message:

```bash
core 16-2.48.2.1 from Canonical✓ installed
```

Get the `core` snap up to date if necessary:

```bash
snap refresh core
```

Since `core` is already up to date, you should see a message:

```bash
snap "core" has no updates available
```

Now use snap to install the `certbot` client for Let's Encrypt:


```bash
snap install --classic certbot
```

The `--classic` option means that `certbot` will not be confined to a restricted environment in the same way as normal snaps are.

The installation ends with a message:

```bash
certbot 1.12.0 from Certbot Project (certbot-eff✓) installed
```

The snap executable for `certbot` is in the directory `/snap/bin`, which should be in your execution PATH. If you want to make extra sure that `certbot` is executable, then also create a symbolic link to `certbot` in your `/usr/bin` directory, which should definitely be in your PATH:

```bash
ln -s /snap/bin/certbot /usr/bin/certbot
```

The symbolic link `/usr/bin/certbot` now points to the actual executable at `/snap/bin/certbot`.

Now you can invoke `certbot` to obtain your SSL certificate from the Let's Encrypt project:

```bash
certbot --nginx 
```

The `--nginx` option tells `certbot` that Nginx is already running and available for use by `certbot` to validate your request.

You are asked to enter your email address in case Let's Encrypt has some urgent notices to send you:

```bash
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel):
```

Type your email address, and press `ENTER`.

You are asked to read and agree to the terms of service:

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o:
```

Type `Y` and press `ENTER`.

You are asked if you want to share your email address with the Electronic Frontier Foundation:

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o:
```

Type `Y` or `N` as you prefer, and press `ENTER`.

You are asked to confirm which fully qualified domain name(s) you want an SSL certificate for:

```bash
Which names would you like to activate HTTPS for?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: your_domain
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
```

Type the number of your server's fully qualified domain name, which would be `1` in our example, and press `ENTER`.

The utility performs "challenges" to confirm that your domain name choice is legitimate. If all is well, you should see some closing messages:

```bash
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/your_domain/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/your_domain/privkey.pem
   Your certificate will expire on 2021-05-15. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le


```

Finally, do a dry run to check that the renewal process will work in 90 days' time:

```bash
certbot renew --dry-run
```

This should end with a message:

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/your_domain/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

## Step 4 — Installing Tor

Install the Tor package:

```bash
apt install tor -y
```

Tor begins running right away. It listens for SOCKS connections on port `9050`.

## Step 5 — Generating Universally Unique Id and Path

In this step, you'll generate a universally unique id (UUID) that will act as a password for your V2Ray server. You'll also generate a random path to which requests are sent in order to access V2Ray instead of Nginx.

Install the UUID package:

```bash
apt install uuid -y
```

You will see the package installation messages. 

Once the install is done, generate a UUID:

```bash
uuid -v 4
```

The option `-v 4` causes the package to create a version 4 UUID. Version 4 UUIDs are completely random and not related to the system time. Since a UUID is composed of 128 random bits, the chances are no one else will ever generate the same UUID.

You will see the sample result below used in the rest of this tutorial:

```bash
e2a48137-a578-4ffa-beed-ca6c99350e0a
```

Also generate a random 8-character path name:

```bash
openssl rand -hex 4
```

The `openssl rand` function generates a pseudo-random number. It will have `4` bytes, which when expressed with the `-hex` option gives 8 hexadecimal characters.

You will see the sample result below used in the rest of this tutorial:

```bash
144c0889
```

## Step 6 — Installing V2Ray

Now you are ready for the install of V2Ray. 

Download the V2Ray installation script:

```bash
wget --no-check-certificate https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh
```

You will see a message saying the script has been saved:

```bash
2021-02-15 13:27:04 (38.3 MB/s) - ‘install-release.sh’ saved [21613/21613]
```

Run the V2Ray installation script:

```bash
bash install-release.sh
```

The script can take a minute or so to run. The install ends with a message:

```bash
Please execute the command: systemctl enable v2ray; systemctl start v2ray
```

You will do these steps in a moment, after you have configured V2Ray.

## Step 7 — Configuring V2Ray and Nginx

In this step, you'll configure V2Ray so that it accepts WebSocket traffic on port `10000`.

Edit the V2Ray configuration file, `/usr/local/etc/v2ray/config.json`:

```bash
nano /usr/local/etc/v2ray/config.json
```

At first, the file contains only opening and closing curly braces:

```bash
{}
```

Delete the initial content.

Copy and paste in the template below:

```
{
  "log": {
    "loglevel": "warning",
    "access": "/var/log/v2ray/access.log", 
    "error": "/var/log/v2ray/error.log"
  },
  "inbounds": [{
    "port": 10000,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "e2a48137-a578-4ffa-beed-ca6c99350e0a",
          "level": 1,
          "alterId": 4
        }
      ]
    },
    "streamSettings": {
      "network": "ws",
      "wsSettings": {
        "path": "/144c0889"
      }
    }
  }],
  "outbounds": [{
    "protocol": "socks",
    "settings": {
      "servers": [{
        "address": "127.0.0.1",
        "port": 9050,
        "auth": "noauth"
      }]
    }
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}

```

There is one inbound, which handles Vmess input on port `10000`. The traffic arrives as WebSocket packets requesting path `/144c0889`. The outbound is in SOCKS format to port `9050`. That is where Tor is listening.

When you've pasted in the template, customize it for your situation by making these changes:

- Replace the UUID `e2a48137-a578-4ffa-beed-ca6c99350e0a` by your own UUID
- Replace the path `/144c0889` by your own random path name

Once you've made the required changes, write the file out to disk, and exit the editor.

Now edit your Nginx configuration file, `/etc/nginx/sites-available/default`. 

```bash
nano /etc/nginx/sites-available/default
```

Make Nginx pass requests for the secret path to V2Ray, which is listening on localhost port `10000`. Inside the `server` block that is listening for `443 ssl`, insert a new `location` block based on the following template:

```bash
location /144c0889 {
	proxy_redirect off;
	proxy_pass http://127.0.0.1:10000;
	proxy_http_version 1.1;
	proxy_set_header Upgrade $http_upgrade;
	proxy_set_header Connection "upgrade";
	proxy_set_header Host $http_host;
	# Show real IP if you enable V2Ray access log
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Replace the path `/144c0889` in the template above by your own secret path. 

Write the file out to disk, and exit the editor.

## Step 8 — Starting V2Ray and Restarting Nginx

Now that your configuration file is correct, start the V2Ray service:

```bash
systemctl enable v2ray
systemctl start v2ray
```

Confirm that Xray is running with the new configuration with the command:

```bash
systemctl status v2ray
```

You should see a status of `active (running)`:

```bash
v2ray.service - V2Ray Service
     Loaded: loaded (/etc/systemd/system/v2ray.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/v2ray.service.d
             └─10-donot_touch_single_conf.conf
     Active: active (running) since Mon 2021-02-15 13:36:35 UTC; 36s ago
       Docs: https://www.v2fly.org/
   Main PID: 3260 (v2ray)
      Tasks: 7 (limit: 1136)
     Memory: 54.6M
     CGroup: /system.slice/v2ray.service
             └─3260 /usr/local/bin/v2ray -config /usr/local/etc/v2ray/config.json

Feb 15 13:36:35 v2ray systemd[1]: Started V2Ray Service.
Feb 15 13:36:35 v2ray v2ray[3260]: V2Ray 4.34.0 (V2Fly, a community-driven edition of V2Ray.) Custom (go1.15.6 linux/amd64)
Feb 15 13:36:35 v2ray v2ray[3260]: A unified platform for anti-censorship.
Feb 15 13:36:35 v2ray v2ray[3260]: 2021/02/15 13:36:35 [Info] v2ray.com/core/main/jsonem: Reading config: /usr/local/etc/v2ray/config.json
```

Restart the Nginx service:

```bash
systemctl restart nginx
```

Confirm that Nginx is running with the new configuration with the command:

```bash
systemctl status nginx
```

Again, you should see a status of `active (running)`:

```bash
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2021-02-15 13:37:57 UTC; 21s ago
       Docs: man:nginx(8)
    Process: 3284 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 3296 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 3298 (nginx)
      Tasks: 3 (limit: 1136)
     Memory: 3.5M
     CGroup: /system.slice/nginx.service
             ├─3298 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             ├─3299 nginx: worker process
             └─3300 nginx: worker process

Feb 15 13:37:57 v2ray systemd[1]: Starting A high performance web server and a reverse proxy server...
Feb 15 13:37:57 v2ray systemd[1]: Started A high performance web server and a reverse proxy server.
```

At this point:

- Nginx is listening on ports `80` and `443`
- V2Ray is listening on port `10000`
- Tor is listening on port `9050`

In the case of a request for the secret path, Nginx will pass the request to V2Ray on port `10000`. If the UUID is correct, V2Ray will pass the request to Tor on port `9050`.

If you logged in as a non-root user and switched to root, you will have to `exit` your switched user session before you can exit the SSH session.

```bash
exit
```

You can then `exit` your SSH session with the server:

```bash
exit
```

## Step 9 — Testing with Client

Now work on your client computer. Before you go any further, test your camouflage web server:

- In an ordinary browser, visit your fully qualified domain name on HTTP (`http://your_domain`). You should see the `Welcome to nginx!` page.
- Again in an ordinary browser, visit your fully qualified domain name on HTTPS (`https://your_domain`). You should see the `Welcome to nginx!` page.

Now you know your camouflage web server is working, you can carry on.

The Qv2ray client is available for Linux, Windows, or macOS. For the rest of this section, we will give instructions for Ubuntu Linux. 

Download the most recent V2Ray core from https://github.com/v2fly/v2ray-core/releases. The V2Fly project is the successor to V2Ray. Your download will have a name that looks like `Xray-linux-64.zip`. In your terminal, unzip the download:

```bash
unzip ~/Downloads/v2ray-linux-64.zip -d ~/Downloads/v2ray-core
```

The contents of the extracted directory `~/Downloads/v2ray-core` are displayed as the extraction proceeds:

```bash
Archive:  /home/amy/Downloads/v2ray-linux-64.zip
   creating: systemd/
   creating: systemd/system/
  inflating: systemd/system/v2ray@.service  
  inflating: systemd/system/v2ray.service  
  inflating: config.json             
  inflating: geoip.dat               
  inflating: geosite.dat             
  inflating: v2ray                   
  inflating: v2ctl                   
  inflating: vpoint_socks_vmess.json  
  inflating: vpoint_vmess_freedom.json
```

Make a directory to hold the core:

```bash
mkdir -p ~/.config/qv2ray/vcore/
```

Copy the contents of the extracted V2Ray core into the new directory:

```bash
cp -rf ~/Downloads/v2ray-core/* ~/.config/qv2ray/vcore/
```

Also download the most recent version of the Qv2ray client from https://github.com/Qv2ray/Qv2ray/releases. The Qv2ray client for Linux is provided as an `AppImage` file. An `AppImage` contains the program, its dependencies, and all files necessary for the program to work, in one self-contained file. Your download will have a name that looks like `Qv2ray.v2.7.0-pre2.linux-x64.AppImage`.

Make Qv2ray executable and start the program. From **Files**:

1. Right-click on the `AppImage` file
2. Select **Properties**
3. Select the **Permissions** tab
4. Check the box **Allow executing file as program**
5. Close the **Properties** dialog box
6. Double-click on the `AppImage` file to launch the program

![Qv2Ray on Ubuntu Linux](/images/qv2ray-linux-new.png)

Press the **Preferences** button, and on the **Kernel Settings** tab, specify your core folder and executable.

![Qv2Ray on Ubuntu Linux Kernel Settings](/images/qv2ray-linux-kernel-settings.png)

Now that the kernel settings are configured, you can add your server configuration to the Qv2ray client.

1. Select the **Default Group** and click the **New** button.
2. For **Host**, type *your_domain* (i.e., your server's fully qualified domain name).
3. For **Port**, type `443`.
4. For **Type**, select **VMess**.
5. Under **Outbound Settings**, for **UUID** put your universally unique id (`e2a48137-a578-4ffa-beed-ca6c99350e0a` in the examples in this tutorial).
6. Set **Alter ID** to `4` to match the server settings.
7. Under **Stream Settings**, select the **Protocol Settings** tab.
8. Set **Transport Protocol** to **ws**.
9. For **Path**, type `/144c0889` or whatever your path is.
10. Still under **Stream Settings**, select the **TLS Settings** tab.
11. Set **Security Type** to **TLS**.
12. Set **Server Address (SNI)** to *your_domain* (i.e., your server's fully qualified domain name).
13. Click **OK**.

![Qv2Ray on Ubuntu Linux Connection Settings](/images/qv2ray-linux-connection-settings.png)

Now select your server under the **Default Group**. 

Click the Connect icon for your server. Notifications appear in Ubuntu to say you are connected.

In your browser, visit https://check.torproject.org to confirm that your web browsing is now passing through the Tor network.

![Check Tor connection on Ubuntu Linux](/images/qv2ray-linux-check-tor.png)

When you have finished browsing, go back to the Qv2ray client. 

Select your server and click the Disconnect icon.

## Conclusion

You now have a working V2Ray Tor on-ramp that you've tested with the Qv2ray client.

You can read more about V2Ray's capabilities at https://www.v2fly.org/en_US/. You can read the Qv2ray documentation at https://qv2ray.net/en/.