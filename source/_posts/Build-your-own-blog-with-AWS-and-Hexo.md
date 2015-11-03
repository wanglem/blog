title: Build your own blog with AWS and Hexo
date: 2015-11-02 15:25:15
tags: website, aws, ec2, hexo, blog
---

Before start, I should really say this is not the best way to create a blog. There are other easy ways, or more powerful ways. This is just me trying something new, and most importantly, have fun.

# Prerequisite
Linux, Git, Node.js, CSS

# Get Your Domain
First you need a personal domain. There are bunch of places, I got mine from [GoDaddy](https://www.godaddy.com).

If you find your preferable doamin with `.com` suffix is not available, try `.net`, `.io`, `.me`, but **NOT** `.gov`, `.biz`, `.info` which are pretty inappropriate.

After you got it, you can go to the manage page, and set "Forwarding" to your facebook page or twitter page, just for fun.

# Setup Your AWS EC2
## Create An EC2 Instance
If you know how to launch a EC2 instance, feel free to skip this section. Continue if you don't.

Login to [Amazon AWS](https://aws.amazon.com), click **My Account** -> **AWS Management Console**, and select EC2.

Click **Launch Instance** (Choose your fav image, I use Amazon Linux).
{% asset_img launch_instance.png %}
{% asset_img select_instance.png %}

Then you'll see the instance type page, remember to select the **t2.micro** type, it is a tiny machine but good enough for your blog. 
{% asset_img review_and_launch.png %}

### Create a security group
On the review page, aws would probably assign you a default Security Group for you instance: `launch-wizard-1`, it's recommended to create your own. Click **Edit Security Groups** and create yours.
{% asset_img edit_security_group.png %}
{% asset_img create_security_group.png %}

### Create a key pair
First time user would need create a key-pair(private key) in order to have ssh access your EC2 instance.
{% asset_img create_key_pair.png %}

Give it a name, download the pem file, and put it into `~/.ssh/`. Remember to rename it to `[your_key_pair].pem`.

Now you can view your running instances:
{% asset_img running_instances.png %}

### Hop into your EC2
First you need find your public dns name from instance page. 
On you terminal: 
```
ssh -i ~/.ssh/[your_key_pair].pem ec2-user@[public_dns_name]
```

If you want to go fancier, add the following to your ~/.ssh/config:
```
Host blog
        hostname [Public DNS Name]
        User ec2-user
        IdentityFile "~/.ssh/[your_key_pair].pem"
```
Then you can run `ssh blog`.

For more info, see [Connect to Your Instance](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-connect-to-instance-linux.html)

## Install nginx
Hop onto your EC2 server and run:
```
sudo yum install nginx
```

Edit your nginx config to change your root:
```
sudo vim /etc/nginx/nginx.conf

make this change:
root     /usr/local/blog
```

Now start your web server:
```
sudo /etc/init.d/nginx start
```

# Intro to Hexo
[Hexo](https://hexo.io) is a lightweighted blog framework. It's [nodejs](https://nodejs.org/en/) backbone makes it super fast. It's using [Markdown Languange(.md)](https://en.wikipedia.org/wiki/Markdown) for editing, if you are comfortable with writing README files, you'll love it's simplicity.

Keep in mind hexo is a simple framework, if you need more dynamic stuff, choose [Wordpress](https://wordpress.org), [Drupal](https://www.drupal.org) or [Joomla](https://www.joomla.org), all FREE.

The best feature I like hexo is it's Markdown support, which is super easy to learn, and it's super straightforward deploy/release/publish process.

## Install and Setup Hexo
Their website has pretty good step-by-step doc, checkout: 
[Hexo Installation Guide](https://hexo.io/docs/)
[Hexo Setup Guide](https://hexo.io/docs/setup.html)

One thing worth mention is to configure the deployment process with your EC2 instance. In this case go to you blog folder, open `_config.yml` and add the following:
```
deploy:
- type: rsync
  host: blog
  user: ec2-user
  root: /usr/local/blog
  port: 22
```

With that, when you run `hexo release` it will publish generated static files to the EC2 instance. Note the attribute `host: blog` was created in your ~/.ssh/config already.

You can also checkout my Github page to view my configuration: https://github.com/wanglem/blog.


# Point Your Domain to Your EC2 Instance
Login to your [GoDaddy](www.godaddy.com) account. Go to you domain detail page, and click on **DNS ZONE FILE**. Under **Host**, edit it to point to you EC2 public IP Address (you can find it on your EC2 instance page).

This might take a few minutes to take effect. 

I'm using godaddy so I'll talk about how to do it there, if you are using other domain provider, google a solution or contact their customer service.

# Trouble Shoot
**Q:** Can not find your public DNS and public IP showing up on your quick view page.
**A:** Go to **Services** -> **VPC**, Select the VPC connected to you EC2, Edit **Summary** -> **Change DNS hostname: Yes**.

**Q:** Unable to install `awscli` using pip on Mac OS El Capitan.
**A:** This is caused by the new security measure called [System Integrity Protection](https://en.wikipedia.org/wiki/System_Integrity_Protection) on El Capitan. Try run
```
pip install --ignore-installed six
```
This is a known bug, see issue tracking on [pip Issue Tracking](https://github.com/pypa/pip/issues/3165)