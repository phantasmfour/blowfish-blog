+++
title = "Self Hosting Blog"
date = "2023-02-26T01:34:33"
slug = "self-hosting-blog-2"
draft = false
+++

This article is old since I have moved to Hugo!!

Why not write my first blog post about creating the blog?

Ghost
-----

Like everyone else I generally setup standard Wordpress sites but I usually like to use some turnkey solution or follow a guide on setting up Wordpress. Doing some searching I came upon [Ghost](https://ghost.org/).

After reading the Ghost installation [docs](https://ghost.org/docs/install/ubuntu/) it seemed easier than Wordpress installs so I started setting it up.

I spun up a VM in my DMZ and followed the installation guide which is very easy to follow. Uninstall is easy and they give you a binary(ghost) which you are able to change configs and restart the service with.  
The only part of the installation that was troublesome was the blog url and the SQL DB. I ended up going with my root domain rather than a subdomain for the blog. The SQL section seemed to indicate to just use root for the install. I setup SQL with a specific user and gave it full permissions for its DB. They don't list specific permissions you need to allow the SQL user but I did not dial back any permissions to test. Its mostly a compromise on the easy install that you are not clued into a lot of the details. I am ok with this compromise currently.  
I setup NGINX without SSL because I was planning to use Cloudflare tunnels to connect back to the site. I ran into a bit of trouble with this as some of my images were loading in via http and some were load via https which were getting blocked via a mixed-content block. This caused some issues and forced me to use https as my URL in ghost config. I was able to leave Always Https and Automatic HTTP Rewrites in Cloudflare and this all seems to play nice so far.

![](/images/first_image_self_hosting.png)I made some fun Nginx configs for Ghost


```
location ^~/ghost {
        allow 10.0.0.0/24;
        deny all;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_pass http://127.0.0.1:2368;
    }
```
This block here is to deny everyone but my local subnet to /ghost. This path is like wp-admin for Wordpress. Nginx regexs actually take precedence over others so when this URL path is determined it overrides other permits. You can also deny this in WAF settings but I went the nginx way since they don't have an option to just tarpit a user which I would have liked. Anything else gives someone an indication a page is there.Then just ontop of the config above I added


```
error_page 403 404 /404.html;

    location = /404.html {
    internal; #return 404
    }
```
This returns a 404 for any 403 error pages. 403 will let someone know that they are forbidden and there is something there. But the 404 does not indicate something exists. I know by telling you I am doing this and showing that /ghost exists its not helping me but its an easy way to add an extra annoyance Cloudflare Tunnels

------------------

Cloudflare Tunnels are the main reason I wanted to do this in general. The main benefit I see people talk about with Cloudflare Tunnels are they let you get from the Internet to your infrastructure even with CGNAT. Basically you run an agent on a system in your local network and you are able to expose anything to the Internet. No port forwarding/VIPs or inbound firewall rules from the internet needed. You are also protected by Cloudflares services and no one sees your public IP. And lastly you do not have to mess with setting up a certificate or SSL settings as you are reverse proxied via Cloudflare.  
This is the main benefit for me, having Cloudflare as a CDN and not needed inbound firewall access is a huge win for me. And also not having to setup a certificate on my side takes out even more hassle. Plus it is all free.  
The only downside to this is trusting Cloudflare. They basically have access into your local network. I think the benefits outweigh the costs. Cloudflare is reputable and I am also keeping non essential data within a DMZ running the tunnel that has no access back into my internal environment so I mitigate a lot of the risk.  
They do not say specifically how Cloudflare gets the data but I assume its just a tunnel that is open between the two hosts and Cloudflare just sends a request for data over that existing tunnel session. In this article they explain a bit more on how they improved tunnel lifetime to make this work even better  
<https://blog.cloudflare.com/argo-tunnels-that-live-forever/>

The Cloudflare [guide](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-guide/local/#set-up-a-tunnel-locally-cli-setup) on setting up the tunnel makes it super simple. I spun up an Ubuntu container via Proxmox in my DMZ VLAN and was able to run a few commands to get it up and running. After that you can set your tunnel to run as a service so that it comes up on reboots. As long as that agent has connectivity to the endpoint you want to host you are set.  


One thing I did uniquely was that I am not using my root domain for anything so I used it for the blog. Cloudflare tunnels basically work off CNAMEs pointing you CNAME you create for the tunnel to Cloudflare. Cloudflare has another feature called CNAME Flattening.   
The DNS RFC(1034) states that CNAMEs must be alone in DNS and also that your root domain must have an SOA and NS record. These two contradict each other an make it so that you should not have a CNAME as your root domain. You can put a CNAME as your root domain but, some of the time doing this you can run into errors because you are violating the RFC and not all applications are programmed to handle this.  


Cloudflare created a way to have RFC complaint CNAMEs for your root domain. As long as Cloudflare is the authoritative DNS server for your domain this will work. When they get a request for the root record they become a DNS resolver and just recurse the CNAME to get an A record. They then just return that A record and make your DNS record look completely normal from the outside. As a side beneift of having Cloudflare do the CNAME chain resolution is that they cache responses and decrease the time for resolution. It also can obscure the fact that you are using Cloudflare since you are getting back a normal IP and not having a CNAME. Full article from Cloudflare here <https://blog.cloudflare.com/introducing-cname-flattening-rfc-compliant-cnames-at-a-domains-root/>

![](/images/cloudflareDNS_second_sef_hosting.png)DNS Record with my root domain as my CNAME![](/images/dnsDigPhantasmfour-1_self_hosting.png)Dig result of my root domain looks normalCloudflare tunnels get even cooler with allowing other protocols like SSH, RDP and SMB over the tunnels. I don't have a usecase for them yet but it seems like they will only keep adding new features.


