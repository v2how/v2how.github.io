---
title: How to Create an Xray VLESS XTLS Server on Ubuntu 20.04
date: 2021-02-12
tags: ["xray", "vless", "server", "ubuntu"]
---

### Introduction

In this tutorial you'll learn how to install Xray on an Ubuntu 20.04 server. 

You'll see how to create a camouflage web server with Nginx, and then you'll see how to install and configure Xray for the VLESS protocol with XTLS security. You'll then test your server with the Qv2ray graphical user interface.

At the end of this tutorial, you'll have a working and tested Xray server.

## Prerequisites

Before you begin this tutorial, you will need:

- A virtual private server (VPS) running Ubuntu 20.04. You can rent servers from providers such as [Google Cloud](https://cloud.google.com) or [Vultr](https://www.vultr.com).
- Your own domain name, which may be [free](https://www.freenom.com) or [paid](https://www.namesilo.com).
- A DNS type `A` record pointing from the fully qualified domain name of your server to the server's IP address. Usually you set up the DNS record at your domain name registrar, but you can also set up DNS records elsewhere, such as at [Cloudflare](https://www.cloudflare.com). If you are using Cloudflare, note that some Xray configurations require DNS services only and will not work with proxying.

## Step 1 — Logging In as Root

SSH into your server. On Linux and macOS, you can use the terminal command SSH to reach your server On Windows, you can either use [PowerShell](https://docs.microsoft.com/en-us/powershell) or a graphical user interface (GUI) such as [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty) or [XSHELL](https://www.netsarang.com).

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
ufw allow from 12.12.12.12/32 to any port 22
```

When you execute that command, you will see a message:

```bash
Rule added
```

If limiting access to a single IP address is not possible, but you can guarantee you will always connect from a known IP address range, then limit SSH access to just that range. For example, if you are in an office which always uses the IP address range `12.12.12.0/24`, then issue the command:

```bash
ufw allow from 12.12.12.0/24 to any port 22
```

If limiting access to an IP address range is not possible, then you will have to open the SSH port for access from the entire world:

```bash
ufw allow ssh
```

In any case, we recommend that you further protect port `22` by using SSH key authentication in preference to password authentication. SSH key pair generation and upload to your server are out of scope for this tutorial. You can find tutorials on this subject on the web.

Open ports `80` (the HTTP port) and `443` (the HTTPS port). These will be used for Xray and for the Nginx camouflage website:

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
OpenSSH                    ALLOW       Anywhere
Nginx HTTP                 ALLOW       Anywhere
22                         ALLOW       12.12.12.0/24
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx HTTP (v6)            ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
443/tcp (v6)               ALLOW       Anywhere (v6)
```

## Step 3 — Creating a Camouflage Website

Now you'll install Nginx and obtain an SSL certificate to camouflage your Xray server.

Install the Nginx web server package:

```bash
apt install nginx
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

Write the file to disk, and exit the editor.

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

The snap executable for `certbot` is in the directory `/snap/bin`, which should be in your execution PATH. If you want to make extra sure that `certbot` is executable, then create a symbolic link to `certbot` in your `/usr/bin` directory, which should definitely be in your PATH:

```bash
ln -s /snap/bin/certbot /usr/bin/certbot
```

The symbolic link `/usr/bin/certbot` now points to the actual executable at `/snap/bin/certbot`.

Now you can invoke `certbot` to obtain your SSL certificate from the Let's Encrypt project:

```bash
certbot certonly --nginx 
```

The `certonly` option means that you just want a certificate, with no further changes to your system. The `--nginx` option tells `certbot` that Nginx is already running and available for use by `certbot` to validate your request.

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
   Your certificate will expire on 2021-05-12. To obtain a new or
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

## Step 4 — Generating Universally Unique Id and Path

In this step, you'll generate a universally unique id (UUID) that will act as a password for your Xray server.

Install the UUID package:

```bash
apt install uuid
```

You will see the package installation messages. 

Once the install is done, generate a UUID:

```bash
uuid -v 4
```

The option `-v 4` causes the package to create a version 4 UUID. Version 4 UUIDs are completely random and not related to the system time. Since a UUID is composed of 128 random bits, the chances are no one else will ever generate the same UUID.

You will see the sample result below used in the rest of this tutorial:

```bash
65183eac-c09f-4fe7-8f0f-95f55281cbb4
```

## Step 5 — Installing Xray

Now you are ready for the install of Xray. Install the prerequisite package `curl`:

```bash
apt install curl
```

It will probably already be present on your Ubuntu system. If so, you will see a message:

```bash
curl is already the newest version (7.68.0-1ubuntu2.4).
```

Copy and paste the command below to download and run the installation script:

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install -u root
```

The `-u root` option is to set up an installation running as `root`. This is required so that Xray can access the SSL certificate private key.

The script takes a few minutes to run. It should end with confirmation messages like this:

```bash
info: Xray v1.2.4 is installed.
You may need to execute a command to remove dependent software: apt purge curl unzip
Created symlink /etc/systemd/system/multi-user.target.wants/xray.service → /etc/systemd/system/xray.service.
info: Enable and start the Xray service
```

Confirm that Xray is running with the command:

```bash
systemctl status xray
```

You should see a result that looks as below:

```
xray.service - Xray Service
     Loaded: loaded (/etc/systemd/system/xray.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/xray.service.d
             └─10-donot_touch_single_conf.conf
     Active: active (running) since Sun 2021-02-12 09:05:34 UTC; 1min 21s ago
       Docs: https://github.com/xtls
   Main PID: 2565 (xray)
      Tasks: 6 (limit: 1137)
     Memory: 3.1M
     CGroup: /system.slice/xray.service
             └─2565 /usr/local/bin/xray run -config /usr/local/etc/xray/config.json
```

As you can see from the above status display, the script has placed your Xray configuration file at `/usr/local/etc/xray/config.json`.

If the status display extends over multiple screens, press the `Q` key on your keyboard to quit the status display.

## Step 6 — Configuring Xray

In this step, you'll configure Xray so that it accepts TCP with XTLS security on port `443`. If a wrong UUID is given, Xray will fall back to passing your traffic to Nginx on port `80`.

Edit the Xray configuration file, `/usr/local/etc/xray/config.json`:

```bash
nano /usr/local/etc/xray/config.json
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
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "65183eac-c09f-4fe7-8f0f-95f55281cbb4",
                        "flow": "xtls-rprx-direct",
                        "level": 0,
                        "email": "love@example.com"
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": 80
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/etc/letsencrypt/live/your_domain/fullchain.pem",
                            "keyFile": "/etc/letsencrypt/live/your_domain/privkey.pem"
                        }
                    ]
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```

When you've pasted in the template, customize it for your situation by making these changes:

- Replace the UUID `65183eac-c09f-4fe7-8f0f-95f55281cbb4` by your own UUID
- Optionally replace the identifier `love@example.com`
- In the Let's Encrpyt certificate and key locations, replace *your_domain* by your server's fully qualified domain name

Once you've made all the required changes, write the file to disk, and exit the editor.

## Step 7 — Restarting Xray

Now that your configuration file is correct, restart the Xray service:

```bash
systemctl restart xray
```

Confirm that Xray is running with the new configuration with the command:

```bash
systemctl status xray
```

At this point, Xray is listening on port `443` and Nginx is listening on port `80`.

## Step 8 — Testing with Client

Before you go any further, test your camouflage:

- In an ordinary browser, visit your fully qualified domain name on HTTP (`http://your_domain`). You should see the `Welcome to nginx!` page.
- Again in an ordinary browser, visit your fully qualified domain name on HTTPS (`https://your_domain`). You should see the `Welcome to nginx!` page.

Now you know your camouflage is working, you can carry on.

Download and install the most recent version of the Qv2ray client for Linux, Windows, or macOS from https://github.com/Qv2ray/Qv2ray/releases. For the rest of this section, we will give instructions for Windows. 

Download and unzip the corresponding Xray core from https://github.com/XTLS/Xray-core/releases/tag/v1.3.0. You will typically have five files in the unzipped folder:

- `geoip.dat`
- `geosite.dat`
- `LICENSE`
- `README.md`
- `xray` binary

Create a folder to hold the core. On Windows it should be named `C:\Users\your_user\AppData\Local\qv2ray\vcore`.

Copy the five files into place in the core folder. 

Start Qv2ray. 

![Qv2ray new install](/images/qv2ray-new.png)

Press the **Preferences** button, and on the **Kernel Settings** tab, specify your core folder and executable.

Now that the kernel settings are configured, you can add your server configuration to the Qv2ray client.

1. Select the **Default Group** and click the **New** button.
2. For **Host**, type *your_domain* (i.e., your server's fully qualified domain name).
3. For **Port**, type `443`.
4. For **Type**, select `VLESS`.
5. Under **Outbound Settings**, for **UUID** put your universally unique id (`65183eac-c09f-4fe7-8f0f-95f55281cbb4` in the examples in this tutorial), and for **Flow** select `xtls-rprx-direct`.
6. Under **Stream Settings**, select the **TLS Settings** tab.
7. Set **Security Type** to `XTLS`.
8. Set **Server Address (SNI)** to *your_domain*.
9. Click **OK**.

Now select your server. Click the Connect icon for your server. 

On Windows, by default your Qv2ray proxy server will listen on `127.0.0.1` port `8889`. You can optionally check this in **Settings** &gt; **Network & Internet** &gt; **Proxy** &gt; **Manual proxy setup**.

In your browser, visit https://www.iplocation.net to confirm that your web browsing is now coming from your server location.

When you have finished browsing, go back to your server in Qv2ray. If you have closed the Qv2ray panel, you can open it again by clicking the Qv2ray icon in the system tray at the bottom right of the Windows desktop. Select your server and click the Disconnect icon.

## Conclusion

You now have a working Xray server that you've tested with the Qv2ray client.

You can find more examples of Xray configurations at https://github.com/XTLS/Xray-examples. You can read the documentation for Qv2ray at https://qv2ray.net/en/.