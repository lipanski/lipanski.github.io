---
layout: default
title: "Blocking malicious requests with nginx + ModSecurity"
tags: devops
comments: true
---

## {{ page.title }}

[ModSecurity](https://modsecurity.org/) is a **web application firewall** integrated with Apache and nginx. It can match request information at various stages and throttle or allow/deny requests based on the rules you define. ModSecurity comes with the [OWASP core rule set](https://github.com/SpiderLabs/owasp-modsecurity-crs) but a [paid set of rules](http://modsecurity.org/rules.html) is also available. Integrating your own rules is quite easy.

This post will try to give you an overview of how to **install the ModSecurity nginx module**, how to **configure the module** and, finally, how to **create a rule for blocking a list of mailicous IPs**.

### Installing ModSecurity-nginx (Bash)

For the equivalent Ansible playbook, skip to the next chapter.

In order to install the [ModSecurity-nginx module](https://github.com/SpiderLabs/ModSecurity-nginx) you'll need to:

* install the *libmodsecurity* dependencies
* build and install *libmodsecurity*
* pull the nginx source code for the nginx version that you're currently running
* build ModSecurity-nginx as a dynamic module by using the nginx source code
* include the built module in your `nginx.conf`

The following commands were run on Ubuntu 14.04. Mileage may vary.

First, you'll want to install the dependencies required in order to build *libmodsecurity*:

```sh
sudo apt install \
  git \
  g++ \
  flex \
  bison \
  curl \
  doxygen \
  libyajl-dev \
  libgeoip-dev \
  libtool \
  dh-autoreconf \
  libcurl4-gnutls-dev \
  libxml2 \
  libpcre++-dev \
  libxml2-dev
```

Next, you'll want to pull the *libmodsecurity* code, build and install it:

```sh
cd opt/
git clone --branch v3.0.0 --depth 1 https://github.com/SpiderLabs/ModSecurity.git

cd ModSecurity/
./build.sh
git submodule init
git submodule update
./configure
make
make install
```

As explained [in my previous post](/posts/nginx-dynamic-modules), in order to build an nginx module you'll need to pull in the source code of the nginx version that you're currently running:

```sh
# Identify your current nginx version
nginx -v

# Pull the source code
cd opt/
wget http://nginx.org/download/nginx-[INSERT NGINX VERSION HERE].tar.gz
tar -xzvf nginx-[INSERT NGINX VERSION HERE].tar.gz
```

Afterwards, download the [ModSecurity-nginx module](https://github.com/SpiderLabs/ModSecurity-nginx):

```sh
cd /opt
git clone --branch v1.0.0 --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
```

Enter the directory where you downloaded the nginx source code, build ModSecurity-nginx as a dynamic module and copy it to `/etc/nginx/modules`:

```sh
cd /opt/nginx-[INSERT NGINX VERSION HERE]
./configure --with-compat --add-dynamic-module=/opt/ModSecurity-nginx --with-cc-opt=-Wno-error
make modules
cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules
```

Finally, include the module somewhere towards the beginning of your nginx configuration:

```nginx
# /etc/nginx/nginx.conf

load_module modules/ngx_http_modsecurity_module.so;
```

### Installing ModSecurity-nginx (Ansible)

This is the equivalent of the previous Bash commands in Ansible. It assumes nginx is already installed. It's been tested with Ubuntu 14.04.

{% raw %}
```yml
- hosts: all
  become: true
  vars:
    nginx_modsecurity_branch: v3.0.0
    nginx_modsecurity_nginx_branch: v1.0.0
  tasks:
  - name: install modsecurity dependencies
    apt: name="{{ item }}"
    with_items:
    - git
    - g++
    - flex
    - bison
    - curl
    - doxygen
    - libyajl-dev
    - libgeoip-dev
    - libtool
    - dh-autoreconf
    - libcurl4-gnutls-dev
    - libxml2
    - libpcre++-dev
    - libxml2-dev

  - name: clone the modsecurity repository
    git: repo="https://github.com/SpiderLabs/ModSecurity.git" version="{{ nginx_modsecurity_branch }}" accept_hostkey=yes depth=1 force=yes dest=/opt/ModSecurity

  - name: build and install modsecurity
    shell: "{{ item }}"
    args:
      chdir: /opt/ModSecurity
    with_items:
    - ./build.sh
    - git submodule init
    - git submodule update
    - ./configure
    - make
    - make install

  - name: clone the modsecurity-nginx repository
    git: repo="https://github.com/SpiderLabs/ModSecurity-nginx.git" version="{{ nginx_modsecurity_nginx_branch }}" accept_hostkey=yes depth=1 force=yes dest=/opt/ModSecurity-nginx

  - name: read the nginx version
    command: nginx -v
    register: nginx_version_output

  # nginx writes the version to stderr
  - name: parse the installed nginx version
    set_fact:
      installed_nginx_version: "{{ nginx_version_output.stderr.split('/')[1] }}"

  - name: download and extract the nginx sources for building the module
    unarchive: src="http://nginx.org/download/nginx-{{ installed_nginx_version }}.tar.gz" remote_src=yes dest=/opt creates="/opt/nginx-{{ installed_nginx_version }}"

  - name: configure the modsecurity-nginx module
    shell: ./configure --with-compat --add-dynamic-module=/opt/ModSecurity-nginx --with-cc-opt=-Wno-error
    args:
      chdir: "/opt/nginx-{{ installed_nginx_version }}"

  - name: build the modsecurity-nginx module
    shell: make modules
    args:
      chdir: "/opt/nginx-{{ installed_nginx_version }}"

  - name: copy the module to /etc/nginx/modules
    shell: cp /opt/nginx-{{ installed_nginx_version }}/objs/ngx_http_modsecurity_module.so /etc/nginx/modules
    args:
      creates: /etc/nginx/modules/ngx_http_modsecurity_module.so

  - name: load modsecurity inside nginx.conf
    lineinfile:
      path: /etc/nginx/nginx.conf
      insertbefore: BOF
      line: "load_module modules/ngx_http_modsecurity_module.so;"
```
{% endraw %}

### Configuring ModSecurity-nginx

The [ModSecurity Reference Manual](https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual) provides a good overview of all the options and rules that ship with ModSecurity. Likewise, the [ModSecurity-nginx README](https://github.com/SpiderLabs/ModSecurity-nginx) provides information about using the nginx module.

This part discusses the basic configuration required in order to start writing your own rules.

Inside your nginx site configuration, enable ModSecurity:

```nginx
server {
  # ...

  modsecurity on;

  location / {
    # ...
  }
}
```

Next, inside every `location` block that should apply the ModSecurity rules, enable the rule engine and some handy options, like logging:

```nginx
server {
  # ...

  modsecurity on;

  location / {
    modsecurity_rules '
      SecRuleEngine On
      SecDebugLog /var/log/nginx/modsecurity-debug.log
      SecDebugLogLevel 3
      SecAuditEngine On
      SecAuditLog /var/log/nginx/modsecurity-audit.log
      SecAuditLogParts ABKZ
    ';

    # ...
  }
}
```

The **audit log** will record the requests that matched your rules, while the **debug log** will contain ModSecurity related debug information, like a misconfigured setup.

If you'd like the ModSecurity **audit logs to use JSON**, add `SecAuditLogFormat JSON` to the mix.

Note that ModSecurity **rule sets can also be loaded from a file**:

```nginx
server {
  # ...

  modsecurity on;

  location / {
    modsecurity_rules_file /etc/modsecurity/my_rules.conf;

    # ...
  }
}
```

...where `/etc/modsecurity/my_rules.conf` would look like this:

```
SecRuleEngine On
SecDebugLog /var/log/nginx/modsecurity-debug.log
SecDebugLogLevel 3
SecAuditEngine On
SecAuditLogParts ABKZ
SecAuditLog /var/log/nginx/modsecurity-audit.log
```

### A rule to block IPs based on a list

It's time to write your first custom rule. Let's introduce a blacklist - a file containing a list of IPs (masked or not) that should be denied any requests:

```
# /etc/modsecurity/blacklist.txt

1.2.3.4
5.6.7.8/24
9.10.11.12/16
```

How to maintain this file is left up to the reader. There are [plenty](https://zeltser.com/malicious-ip-blocklists/) of [lists](http://iplists.firehol.org/) out there; pick the one that suits your needs and make sure you strip any additional information or markup, aside from the IPs and their masks.

Once you have your list, add the follwing rule either to your nginx site configuration or to your dedicated ModSecurity configuration file:

```
SecRule REMOTE_ADDR "@ipMatchFromFile /etc/modsecurity/blacklist.txt" id:1,phase:1,deny,status:403,msg:\'blacklist\'
```

A couple of things are worth mentioning here:

* The `ipMatchFromFile` call is one of the [many transformation functions](https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual#Transformation_functions) that you can use to match ModSecurity variables.

* Likewise, `REMOTE_ADDR` is one of the [many variables](https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual#Variables) that you can use to match request details, like the request IP in this case.

* Every rule needs to have a unique `id`.

* The `phase` refers to the [event of the request lifecycle]((https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual#Processing_Phases)) when the rule can be applied: after parsing request headers, after parsing the request body, after parsing the response headers, after parsing the response body, after logging etc. In this case, `phase:1` means we apply our rule when we already have the request headers.

* The `deny` keyword is the **action** that a matched requests will trigger. The opposite would be `allow` (if you'd be building a whitelist). Another interesting action is `drop`, if you'd like to `drop` the TCP connection instantly (like in the case of DDOS attacks), but this value didn't really work in my tests.

* You can set the `status` code that a matched request will receive. The response body will be the nginx template that coresponds to this status code.

* The `msg` option can be used as an human readable identifier which will appear in the audit log.

Once you've added the rule, make sure to reload nginx:

```sh
sudo service nginx reload
```

That's pretty much it. Requests by IPs on your list will now be blocked and clients will see a *403* error.

Note that any changes to your list of IPs or to your ModSecurity rules will require *reloading nginx* in order for ModSecurity to pick up the changes.

### Running nginx behind a reverse proxy

If you're running nginx behind a reverse proxy (e.g. a load balancer), which hides the client IP but sets the `X-Forwarded-For` header correctly, I recommend setting the `real_ip_header` option in your nginx configuration:

```nginx
# /etc/nginx/nginx.conf

http {
  real_ip_header X-Forwarded-For;

  # The proxy address that you trust to set the X-Forwarded-For header correctly
  set_real_ip_from 10.0.0.0/8;
}
```

...which will ensure that the `REMOTE_ADDR` variable in ModSecurity points to the client IP and not the reverse proxy.

Sometimes, you'll also want to enable the `real_ip_recursive` option - see the [documentation](http://nginx.org/en/docs/http/ngx_http_realip_module.html#real_ip_recursive) for more details.

### Alternatives

One alternative that works with nginx is [lua-resty-waf](https://github.com/p0pr0ck5/lua-resty-waf). It requires [OpenResty](https://openresty.org/en/) though or recompiling nginx with OpenResty/Lua.

Another one would be the cloud-based [AWS WAF](https://aws.amazon.com/waf/), which comes with some annoying restrictions: IP lists are limited to 1000 entries and you can only use `/8`, `/16`, `/24` or `/32` CIDR masks. The suggested workaround is to create multiple lists of 1000 entries and convert other masks to the available ones. Another limitation on the AWS WAF is that some AWS regions don't come with all its features (e.g. the load balancer integration), so make sure to check availability in your region before.

If you only want to allow/deny a list of IPs, there's the [nginx-ipset-blacklist module](https://github.com/Vasfed/nginx_ipset_blacklist), but it looks quite outdated and won't work with newer nginx versions. On the other hand, you could use plain `iptables` or `iptables` integrated with [IpSet](http://ipset.netfilter.org/) - a fast lookup store for IP addresses.

### Links

* [Compiling and using dynamic nginx modules](/posts/nginx-dynamic-modules)
* [ModSecurity Compilation Recipe for Ubuntu 15.04](https://github.com/SpiderLabs/ModSecurity/wiki/Compilation-recipes#ubuntu-1504)
* <https://modsecurity.org/>
* <https://github.com/SpiderLabs/ModSecurity>
* <https://github.com/SpiderLabs/ModSecurity-nginx>
* <https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual>
* <https://danieljamesscott.org/all-articles/9-articles/34-whitelisting-ips-for-mod-security-when-behind-a-load-balancer.html>
