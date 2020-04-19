#### DOCS:    
1. https://research.kudelskisecurity.com/2017/06/07/installing-wireguard-the-modern-vpn/     
2. https://gist.github.com/nealfennimore/92d571db63404e7ddfba660646ceaf0d    
3. https://angristan.xyz/2019/01/how-to-setup-vpn-server-wireguard-nat-ipv6/    
4. https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8


**NB;**  
- the private IP address `192.168.3.XX` doesn't have to be an IP you own.  
create a new vps/ip on ua cloud provider and check IP location on https://www.whatismyip.com/ip-address-lookup   
it should show location as the location u want.     
- this requires at least ubuntu 19.10

### I. SERVER
```bash
apt -y update && \
apt -y install wireguard
```
```bash
# this will generate server private key & public key
wg genkey | tee ServerPrivatekey | wg pubkey > ServerPublickey
```

`cat /etc/wireguard/wg0.conf`  
```bash
[Interface]
Address = 192.168.3.1/24, fd86:ea04:1115::1/64
ListenPort = 5555
PrivateKey = <ServerPrivatekey>
# the following two lines may not be neccesary
# If you only want to create a tunnel but not forward all your traffic through the server you can skip those.
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE; ip6tables -A FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE; ip6tables -D FORWARD -i wg0 -j ACCEPT; ip6tables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
# eth0 is the servers public interface. You can find what yours is by;
# ip -4 route ls | grep default | grep -Po '(?<=dev )(\S+)' | head -1

[Peer]
PublicKey = <ClientPublickey>
AllowedIPs = 192.168.3.2/32
```
`cat /etc/sysctl.conf`
```bash
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```
`sysctl -p` # to enable packet forwarding


### II. CLIENT
```bash
apt -y update && \
apt -y install wireguard 
# apt -y install openresolv # may be required if wg is unable to start
```

```bash
# this will generate client private key & public key
wg genkey | tee ClientPrivatekey | wg pubkey > ClientPublickey
```
`cat /etc/wireguard/wg0.conf`
```bash
[Interface]
Address = 192.168.3.2/32
ListenPort = 5555
PrivateKey = <ClientPrivatekey>
# or use a dns server from uk; https://public-dns.info/nameserver/gb.html
# or use <ServerPublicIPadress>
DNS = 1.1.1.1
# the following two lines may not be neccesary
PostUp = iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT

[Peer]
PublicKey = <ServerPublickey>
# This can be narrowed down if you only want some traffic to go over VPN.
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <ServerPublicIPadress>:5555
```

### III. START/STOP
```bash
systemctl stop wg-quick@wg0
systemctl start wg-quick@wg0
systemctl status wg-quick@wg0
journalctl -xf -n10 -u wg-quick@wg0.service
sudo wg
```
**NB:** you may have to install `apt-get -y install openresolv` if wire-guard is unable to start

### IV. edit configs   
to edit `/etc/wireguard/wg0.conf` you need to; 
- a. stop wg
- b. edit files
- c. restart wg  

**NB:** edits made while wg is still running may not be persisted   
**NB:** regarding dns leaks. I had a chat on the wireguard irc and;  
```bash

zx2c4:
  on the client, to fix dns leaks you can either
  1) not use debian/ubuntu
  2) add this "kill switch" to your config file:
  PostUp = iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
  PreDown = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT

amdj:
- just settle for confirming that your query is being sent over wireguard and call it a day.
  this is easy with e.g. tcpdump, or you can enforce it with the rules zx2c4 gave you.
- if you're talking to 1.1.1.1 over the wireguard tunnel then as far as cloudflare is concerned they can't see your real address 
  but a leak test is still going to tell you that you have a leak because the query isn't coming from your server (but rather from cloudflare).
- what's ironic is that satisfying the conditions of these tests(dns leak test services) when you're running your own VPN will actually 
  give you less privacy in many circumstances.
- the operators of authoritative nameservers can see the address of the recursor that's asking them questions. 
  if that recursor is running on your endpoint using an IP address registered to you (e.g. in whois data) then you've given your identity 
  away to every domain admin you do lookups for.
```
**NB:** zx2c4 is main author of wireguard   
