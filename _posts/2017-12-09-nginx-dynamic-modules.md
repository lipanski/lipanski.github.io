---
layout: default
title: Compiling and using dynamic nginx modules (ipset blacklist)
---

## {{ page.title }}

Dynamic nginx modules allow you to extend the functionality of nginx without having to recompile the entire suite. However, compiling these modules is not exactly trivial.

First of all, you'll want to pull the source code of the module to the target machine. For this example, I'll be using the `ngx_http_ipset_blacklist` module, available [here](https://github.com/Vasfed/nginx_ipset_blacklist). The following commands will be run as *root*.

```
cd /opt
git clone https://github.com/Vasfed/nginx_ipset_blacklist.git
```

### Converting a static module to a dynamic module

In case the module is static, you'll want to convert its configuration to a dynamic one. You can figure out if a module is static by looking inside its `config` file, present inside the root of the module. If this file contains a call to `auto/module`, then it's dynamic. Otherwise you'll need to convert it.

The `config` file of the `nginx_ipset_blacklist` project looks like this:

```
ngx_addon_name=ngx_http_ipset_blacklist

HTTP_MODULES="$HTTP_MODULES ngx_http_ipset_blacklist"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_ipset_blacklist.c"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ipset_read.c"
```

It's quite easy to convert it to a dynamic module by changing the file to:

```
ngx_addon_name=ngx_http_ipset_blacklist

if test -n "$ngx_module_link"; then
    ngx_module_type=HTTP
    ngx_module_name=ngx_http_ipset_blacklist
    ngx_module_srcs="$ngx_addon_dir/ngx_http_ipset_blacklist.c $ngx_addon_dir/ipset_read.c"

    . auto/module
else
    HTTP_MODULES="$HTTP_MODULES ngx_http_ipset_blacklist"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_ipset_blacklist.c"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ipset_read.c"
fi
```

You can read more about converting static modules to dynamic modules [here](https://www.nginx.com/resources/wiki/extending/converting/).

### Compiling the module

In order to compile the module, you'll need the nginx source code of the very same version that's running on the target machine. You can figure out the version by calling:

```
nginx -v
```

It is highly recommended that you use or upgrade to **nginx 1.11.5 or higher**. This version introduced the `--with-compat` option. It facilitates compiling additional dynamic modules with minimal effort assuming that nginx was originally compiled `--with-compat` as well. For earlier versions, a world of pain and missing dependencies awaits you.

For this example, I'm using nginx 1.12.2. Let's download and unpack the source code:

```
cd /opt
wget http://nginx.org/download/nginx-1.12.2.tar.gz
tar -xzvf nginx-1.12.2.tar.gz
```

Assuming your module lives under `/opt/nginx_ipset_blacklist`, run the following:

```
cd /opt/nginx-1.12.2
./configure --with-compat --add-dynamic-module=/opt/nginx_ipset_blacklist/ --with-cc-opt=-Wno-error
```

The last option `--with-cc-opt=Wno-error` is optional and prevents *converting warnings to errors*. It was required in the case of the `nginx_ipset_blacklist` module, but you might not need it.

Once the `./configure` call succeeded, you can run:

```
make modules
```

This will build your module and place it inside the `objs/` folder. The compiled module bears the `.so` extension. The final step is copying it over to the `/etc/nginx/modules` directory:

```
cp objs/ngx_http_ipset_blacklist.so /etc/nginx/modules
```

In case something broke during the previous steps, you can always clean up the build directory, by calling:

```
make clean
```

...and starting again from the `./configure` step.

### Using a dynamic module

The final step is actually using the module. Open your `/etc/nginx/nginx.conf` and add the following line towards the beginning of the file:

```
load_module modules/ngx_http_ipset_blacklist.so;
```

...and restart nginx:

```
service nginx restart
```

### Links

- <https://github.com/Vasfed/nginx_ipset_blacklist>
- <https://www.nginx.com/resources/wiki/extending/converting/>
- <https://shadrin.org/nginx/blog/content/compiling-dynamic-modules-nginx-plus.html>
- <https://trac.nginx.org/nginx/ticket/1377>
- <https://stackoverflow.com/questions/5451237/nginx-0-9-6-frustrating-compile-issues-ubuntu-gcc-4-6-1>
