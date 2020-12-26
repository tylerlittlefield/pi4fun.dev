+++
author = "Tyler Littlefield"
title = "👨🏻‍🍳 Self hosting a website"
date = "2020-12-25"
description = "How I deployed a website from my raspberry pi cluster"
tags = [
    "raspberry-pi",
    "website",
]
+++

In this guide, I show you how to deploy a self hosted website using a Raspberry Pi. By the end of this guide you should have:

* A website you can access with `https://your-domain.dev`
* Automatic updates on push events with github actions

A summary of the main tools used are as follows:

* Raspberry Pi, the machine to actually server the site
* Google domains, a service to buy a domain and configure a synthetic DNS record
* ddclient, a client for updating dynamic DNS entries
* hugo, a framework for building static websites
* NGINX, a webserver
* Github actions, to update the site with any commit to git repository

# Disclaimer

I have next to zero experience with this stuff. In the past, I have used github pages, netlify and others to deploy stuff but this is my first time deploying something from my own self hosted server. **If security is a concern**, you should be careful as this guide covers none of that. The only remotely secure thing I have done is use a `.dev` domain which requires `https`.

# Prerequisites

Throughout the rest of this text, I will assume you already have your Raspberry Pi set up, connected to internet, and can SSH into it. Additionally, your router should have ports 443/80/22 open for this device.

# Purchasing a domain

The first step is purchasing a domain name. I have always gone with google domains and I am not sure if it matters. Usually it doesn't, but in this guide I create a **synthetic record** which is unique to google domains:

> Synthetic records are a concept unique to Google Domains. They perform tasks that require multiple resource records, such as mapping the domain and subdomains required to integrate with Google Workspace, creating email forwarding aliases, or even adding all the resource records required to integrate your domain with a 3rd-party web host.

So if you don't want to deviate and figure out the network stuff on your own, go with google domains. In my case, I purchased `pi4fun.dev`, the `.dev` is a recently popular domain that requires `https` be used. From the docs, it says:

> The .dev top-level domain is included on the HSTS preload list, making HTTPS required on all connections to .dev websites and pages without needing individual HSTS registration or configuration. Security is built in.

It's also just a cool domain, so I like using it. After you've purchased a domain, you will need to make a synthetic record

![](/images/synthetic-record.png)

# Network stuff and ddclient magic

I don't understand `ddclient` very well or anything network related for that matter. My poor interpretation is that it's used for services hosted on a dynamic IP, or services whose IP address changes from time to time. I believe this is a tricky problem for inexperienced people like myself. For example, logging into your Pi might work one day, but fail the next as the IP address has changed.

My ISP is google fiber and I have personally never ran into this issue. Google fiber lets you *reserve* IPs by device and I have done this for all 3 of my Pi's. I have no idea if this is the reason my IP's haven't changed for each Pi.

Anyway, SSH into you Pi and install ddclient with:

```bash
sudo apt install ddclient
```

After a GUI will appear and ask you to fill things out. I followed this guide to help me fill things out, the following steps are essentially the same:

* https://engineerworkshop.com/blog/connecting-your-raspberry-pi-web-server-to-the-internet/

In short, you will:

1. Select "other" as the dynamic DNS provider
2. Provide "domains.google.com" as the dynamic DNS server
3. Update the `ddclient.conf` file

If you are nervous about what to fill out, I don't think you need to be as we are going to update the config file anyways. Start by running:

```bash
sudo nano /etc/ddclient.conf
```

And then provide:

```text
daemon=300
protocol=dyndns2
use=web
server=domains.google.com
ssl=yes
login=<insert username from google domains (see screenshot above)>
password='<insert password from google domains (see screenshot above)- KEEP SINGLE QUOTES'
<your domain name here, e.g. pi4fun.dev>
```

Next, the defaults with:

```bash
sudo nano /etc/default/ddclient
```

And modify the file to have:

```text
run_daemon="true"
daemon_interval="300"
```

Finally, start the client:

```bash
sudo systemctl start ddclient
```

Test the configuration with:

```bash
sudo ddclient -daemon=0 -debug -verbose -noquiet
```

If all goes well, you should see a success message. If you run the line once more, you should see:

```bash
SUCCESS:  pi4fun.dev: skipped: IP address was already set
```

Now if you go back to google domains, you should see that the public IP is associated with the synthetic record. It should no longer have a value of `0.0.0.0`. Finally, the last thing we need to configure on google domains is **custom resource records**. We need to add an `A` record so that you can access the site from `www.pi4fun.dev`, not just `pi4fun.dev`.

![](/images/custom-resource-record.png)

# Pick a hugo theme

The next step is to pick a hugo theme. If you want to build your own site, without a theme you can skip this part. I have a lot of experience with R programming, so I have always used `blogdown` to create an R project that I can use to develop my site and track in git. I start by running the following commands:

```r
# set up the project
blogdown::new_site("project-name", theme = "user/repo")

# use git, assumes you have github credentials in your .Renviron file
usethis::use_git()

# push to github
usethis::use_github()

# use github actions
usethis::use_github_actions()
```

By default, `usethis::use_github_actions()` will create and R CMD Check action, used for checking an R package. Rename this to something else like `Deploy.yaml` and provide the following action:

```yaml
name: Deploy
on: [push]
jobs:

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
    - name: Copy files via ssh key
      uses: appleboy/scp-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.KEY }}
        source: "public/*"
        target: ${{ secrets.TARGET }}
        rm: true
```

The goal is to "deploy" our app by using `scp` to essentially copy all the contents of the `public` folder from our site to the server on every push. With this repository opened in a browser, add the secrets:

![](/images/gh-secrets.png)

* `HOST` is the hosts IP, the public IP associated with the synthetic record
* `KEY` is the SSH key used for logging into the Raspberry Pi, you can copy it with `pbcopy < ~/.ssh/id_rsa` assuming your key is stored at this path
* `TARGET` is the directory you want to copy the files to, you could probably just provide this instead of making it a secret if you want
* `USERNAME` is the username you use to login to the Raspberry Pi

With these secrets stored, every push should trigger an update to your site as it will remove the old contents and copy the new.

# Configuring NGINX

At this point, you should have:

1. A domain that we can point to over the internet
2. A website that automatically pushes the contents of the `public` folder on push

However, we still need to:

1. Configure NGINX
2. Add a certificate so that `https` works

Start by logging into the Pi and installing NGINX with:

```bash
sudo apt install nginx
```

At this point, NGINX server is running and will come back to it later to point to our sites contents. Now let's use certbot to add a LetsEncrypt certificate that automatically renews. Navigate to https://certbot.eff.org/ and specify your environment. In my case, I'm using NGINX and Ubuntu. I will skip over the code needed to run here, the certbot documentation is very clear, concise, and up to date. 

After, expand the certificate to include `www.pi4fun.dev` with:

```bash
certbot certonly --standalone -d pi4fun.dev -d www.pi4fun.dev --dry-run
```

If the dry run passes, remove the `--dry-run` flag and rerun. After this, the last step is to point NGINX to the files your site uses. This directory will be the one that the github action is pointing to, i.e. the `TARGET` environment variable. Edit the config file with:

```bash
nano /etc/nginx/sites-available/default
```

And modify like so:

```text
# comment out the default root and specify the new root
# root /var/www/html;
root /path/to/your/site/content/public;
```

Do this for both server blocks. Save the file and check if the syntax is good:

```bash
nginx -t
```

If it passes, restart NGINX:

```bash
systemctl restart nginx
```

After this, we are all done! You should now have a site that is fully functional, and self hosted from your Raspberry Pi.