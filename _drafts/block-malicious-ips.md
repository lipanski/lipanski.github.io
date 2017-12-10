Note: block only port 80 + always whitelist own IPs (just in case) + https://www.netfilter.org/documentation/HOWTO/netfilter-extensions-HOWTO-3.html#ss3.18 + https://unix.stackexchange.com/questions/263579/is-there-a-local-firewall-to-block-by-x-forwarded-for-ips-behind-the-reverse

```
sudo su
apt install ipset

# Create set
ipset create spamhaus hash:ip

# Add IPs to the set
ipset add spamhaus 1.2.3.4/32

# New
iptables -N spamhaus

# Flush
iptables -F spamhaus

# Append set
iptables -A spamhaus -m set --match-set spamhaus src -j DROP

# Insert rule into chain
iptables -I INPUT ! -i lo -j spamhaus

# List rules
iptables -nL INPUT
```


```
ipset destroy torexits-update
ipset create torexits-update iphash
curl "https://check.torproject.org/cgi-bin/TorBulkExitList.py?ip=176.58.108.237&port=443" > /tmp/torexits
for i in $( egrep -v "^#" /tmp/torexits ); do ipset add torexits-update $i; done
ipset swap torexits-update torexits
ipset destroy torexits-update
```

Links:

- https://www.spamhaus.org/drop/
- https://files.pfsense.org/lists/fullbogons-ipv4.txt
- https://www.cyberciti.biz/faq/linux-iptables-drop/
- http://ipset.netfilter.org/ipset.man.html
- https://www.cyberciti.biz/faq/iptables-block-port/
- http://blog.ls20.com/securing-your-server-using-ipset-and-dynamic-blocklists/
- https://bugs.launchpad.net/ufw/+bug/1571579
- https://spielwiese.la-evento.com/xelasblog/archives/74-Ipset-aus-der-Spamhaus-DROP-gemeinsam-mit-ufw-nutzen.html
