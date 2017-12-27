### Install

```
sudo su

# Install libmodsecurity dependencies
apt install g++ flex bison curl doxygen libyajl-dev libgeoip-dev libtool dh-autoreconf libcurl4-gnutls-dev libxml2 libpcre++-dev libxml2-dev

# Build and install libmodsecurity
cd opt/
git clone --branch v3.0.0 --depth 1 https://github.com/SpiderLabs/ModSecurity.git
cd ModSecurity/
./build.sh
git submodule init
git submodule update
./configure
make
make install

# Build and install the nginx modesecurity module
cd /opt
git clone --branch v1.0.0 --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git
wget http://nginx.org/download/nginx-1.12.2.tar.gz
tar -xzvf nginx-1.12.2.tar.gz
cd /opt/nginx-1.12.2
./configure --with-compat --add-dynamic-module=/opt/nginx_ipset_blacklist/ --with-cc-opt=-Wno-error
make modules
cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules
```

### Use

Add the following line to /etc/nginx/nginx.conf:

```
load_module modules/ngx_http_modsecurity_module.so;
```

Add the following to you site nginx configuration:

```
modsecurity on;

location / {
  modsecurity_rules '
    SecRuleEngine On
    SecRule REMOTE_ADDR "@ipMatchFromFile /opt/ips.txt" id:1,phase:1,deny,status:429,msg:\'blocklist\'
    SecRule REQUEST_HEADERS:X-Forwarded-For "@ipMatchFromFile /opt/ips.txt" id:2,phase:1,deny,status:429,msg:\'blocklist\'
    SecDebugLog /var/log/nginx/modsecurity-debug.log
    SecDebugLogLevel 3
    SecAuditEngine On
    SecAuditLogParts ABZ
    SecAuditLog /var/log/nginx/modsecurity-audit.log
    SecAuditLogFormat JSON
  ';
}
```

The status code of the rule will use the corresponding nginx template. If you want to customize the respone body, just update the template.

```yaml
vars:
  nginx_modsecurity_dir: /opt/ModSecurity
  nginx_modsecurity_branch: v3.0.0
  nginx_modsecurity_nginx_branch: v1.0.0
tasks:
- name: install modsecurity dependencies
  apt: name="{{ item }}"
  with_items:
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
```

Python script:

```python
#!/usr/bin/env python

import httplib
import re
import argparse

# Edit this
SOURCES = [
    ['example.com', 'list.txt', True]
]

IP_REGEX = re.compile(r'\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}(?:/\d{1,2}|)')

def fetch_ips(host, path, https=True):
    ips = []

    if https:
        connection = httplib.HTTPSConnection(host)
    else:
        connection = httplib.HTTPConnection(host)

    connection.request('GET', path)
    response = connection.getresponse()
    if response.status == 200:
        for line in response.read().splitlines():
            ips += IP_REGEX.findall(line)

    connection.close()

    return ips

def fetch_all_ips():
    ips = []

    for source in SOURCES:
        ips += fetch_ips(source[0], source[1], source[2])

    return list(set(ips))

def main():
    parser = argparse.ArgumentParser(description='Collects a list of malicious IPS and stores them on disk')
    parser.add_argument('output', type=str, help='The path to the output file')

    args = parser.parse_args()

    file = open(args.output, "w")
    for ip in fetch_all_ips():
        file.write(ip + '\n')
    file.close()

if __name__ == "__main__":
    main()

```

### References

- [ModSecurity Nginx Module](https://github.com/SpiderLabs/ModSecurity-nginx)
- [ModSecurity Compilation Recipe for Ubuntu 15.04](https://github.com/SpiderLabs/ModSecurity/wiki/Compilation-recipes#ubuntu-1504)
- [ModSecurity Reference Manual](https://github.com/SpiderLabs/ModSecurity/wiki/Reference-Manual)
- <https://danieljamesscott.org/all-articles/9-articles/34-whitelisting-ips-for-mod-security-when-behind-a-load-balancer.html>
