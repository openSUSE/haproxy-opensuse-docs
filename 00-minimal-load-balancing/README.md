# Minimal Load Balancing configuration

# Requirements:

- 2 Instances, Virtual Machines or bare-metal. We will use to openSUSE Virtual Machines KVM, but it doesn't matter.
  * 1 will be  the haproxy Load Balancer
  * 1 will be the apache server. 

- I assume the two machine are connected on same network or can communicate each other. 


# HOWTO:

1) Configure apache server
2) Configure haproxy

# 1) apache2 server 

In my conf, it has adress: `10.162.29.89`

* Install with `zypper in apache2`

* Create virtual-host `vim /etc/apache2/conf.d/example.conf`

```
<VirtualHost *:80>

ServerAdmin webmaster@.example.com

DocumentRoot /srv/www/htdocs/example

ServerName www.example.com

ErrorLog /var/log/apache2/example.com-error_log

CustomLog /var/log/apache2/example.com-access_log common

</VirtualHost>
```

* Website/document upload with:

```
mkdir /srv/www/htdocs/example
echo "I am the webserver-apache, welcome" > /srv/www/htdocs/example/index.html
```

* add host entry with 
```
echo "127.0.0.1 www.example.com" >> /etc/hosts
```

* start web-server
```
systemctl restart apache
```

* visiting the page on you web-browser `http://10.162.29.89` ( where the IP is your public IP of server)

# 2) HAPROXY  

In my conf, `10.162.29.126` is the HAPROXY server.

On the `Load Balancer` OS install with

```
zypper in haproxy
```

### Configure

* Edit `/etc/haproxy/haproxy.cfg` 

In my conf:
* `10.15.29.89` is the apache2 server.
* `10.162.29.126` is the haproxy server.

The following doc is mostly the standard one.

```
global
  log 127.0.0.1 local2
  maxconn 32768
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon
  stats socket /var/lib/haproxy/stats user haproxy group haproxy mode 0640 level operator
  tune.bufsize 32768
  tune.ssl.default-dh-param 2048
  ssl-default-bind-ciphers ALL:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!PSK:!RC4:!ADH:!LOW@STRENGTH

defaults
  log     global
  mode    http
  option  log-health-checks
  option  log-separate-errors
  option  dontlog-normal
  option  dontlognull
  option  httplog
  option  socket-stats
  retries 3
  option  redispatch
  maxconn 10000
  timeout connect     5s
  timeout client     50s
  timeout server    450s

listen  stats
        bind *:9001
        stats enable
        stats uri  /monitor
        stats refresh 5s

frontend my-haproxy-fancy-name-here
    bind *:9200
    mode http
    default_backend web_servers

backend web_servers
    balance roundrobin
    server server1 10.162.29.89:80 check
```

Frontend  accepts requests from clients ( so we listen on port 9200, it could be different)

Backend are the webservers, in our case only 1 entry. ( see reference doc a the end for more info)


* start with `systemctl restart haproxy`


`DEBUG TIP`: In case you want to debug something wrong, you can use also this `haproxy -f /etc/haproxy/haproxy.cfg  -V`. This start daemon from your CLI


# Testing your haproxy

* 01) Visit the page on the haproxy ip with your browser `http://10.162.29.126:9001/monitor`
      -> This will provide you `stats`

* 02) Visit haproxy with your browser`http://10.162.29.126:9200/` -> the `haproxy` is redirecting the traffic to `apache` webserver


# References:

For more detailed infos, refer to upstream doc:

* https://www.haproxy.com/de/blog/the-four-essential-sections-of-an-haproxy-configuration/

* http://www.haproxy.org/

* Linux KVM Terraform: https://github.com/dmacvicar/terraform-provider-libvirt if you want to use Linux and terraform Kvm. ( I used this in tutorial)
