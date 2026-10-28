# Networking and troubleshooting cheatsheet

[← Cheatsheet index](README.md)

## Interfaces, routes, DNS

```bash
ip a; ip -br a; ip link set eth0 up; ip addr add 10.0.0.5/24 dev eth0
ip r; ip route get 8.8.8.8; ip route add 10.1.0.0/16 via 10.0.0.1
ip neigh                              # ARP table
ss -tulpn                             # listening sockets with process (replaces netstat -tulpn)
ss -tan state established; ss -s; ss -tnp dst :5432
netstat -rn; route -n                 # legacy
cat /etc/resolv.conf; resolvectl status; /etc/hosts; /etc/nsswitch.conf
dig +short example.com; dig example.com A @8.8.8.8; dig -x 10.0.0.5; dig example.com NS; dig +trace example.com; dig example.com CNAME +short
nslookup example.com; host example.com
hostname -I; hostnamectl
```

## Connectivity tests

```bash
ping -c3 host; ping -M do -s 1472 host              # MTU test (1500 - 28)
traceroute -n host; mtr -rwc 20 host; tracepath host
nc -zv host 443; nc -zvu host 53; nc -l 8080         # listen for test
timeout 3 bash -c '</dev/tcp/host/443' && echo open  # no nc available
curl -v https://host; curl -sI https://host; curl -o /dev/null -s -w '%{http_code} %{time_total}s\n' https://host
curl --resolve host:443:10.0.0.5 https://host/        # bypass DNS, test specific backend
curl -k; curl --cacert ca.pem; curl -H 'Host: x' http://ip/; curl -x http://proxy:3128 https://x
curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' https://x/api
curl -w '@fmt.txt' ...  # dns/connect/tls/ttfb timings
wget -qO- url; wget --spider url
telnet host 25
```

## TLS

```bash
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer
openssl s_client -connect host:443 -showcerts </dev/null       # full chain
openssl x509 -in cert.pem -noout -text; openssl x509 -in cert.der -inform der -out cert.pem
openssl req -new -newkey rsa:2048 -nodes -keyout k.pem -out r.csr -subj '/CN=x'
openssl rsa -in k.pem -check; openssl verify -CAfile ca.pem cert.pem
openssl x509 -noout -modulus -in cert.pem | md5sum ; openssl rsa -noout -modulus -in k.pem | md5sum   # key matches cert?
echo | openssl s_client -connect host:443 2>/dev/null | grep -E 'Protocol|Cipher'
update-ca-trust (RHEL: /etc/pki/ca-trust/source/anchors/) ; update-ca-certificates (Debian: /usr/local/share/ca-certificates/)
```

## Packet capture

```bash
tcpdump -i any -nn port 443; tcpdump -i eth0 -nn host 10.0.0.5 and port 8080
tcpdump -i any -nn -w cap.pcap -c 1000; tcpdump -r cap.pcap
tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn) != 0 and not src net 10.0.0.0/8'
tcpdump -i any -nn -A port 80 | grep Host
tshark -i eth0 -Y 'http.request'
```

## Firewall

```bash
iptables -L -n -v; iptables -t nat -L -n -v; iptables -S
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT; iptables -P INPUT DROP
nft list ruleset
firewall-cmd --list-all; firewall-cmd --add-port=8080/tcp --permanent; firewall-cmd --reload
ufw status; ufw allow 443/tcp; ufw enable
getenforce; setenforce 0; ausearch -m avc -ts recent; semanage port -l | grep http; setsebool -P httpd_can_network_connect 1
```

## SSH

```bash
ssh -i key.pem -J bastion user@private-host           # jump host
ssh -L 8080:internal:80 user@bastion                  # local forward
ssh -R 9000:localhost:3000 user@host                  # remote forward
ssh -D 1080 user@host                                 # SOCKS
ssh -vvv user@host; ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5
ssh-keygen -t ed25519 -C 'me'; ssh-copy-id user@host; ssh-add -l; ssh-keygen -R host   # remove known_hosts entry
~/.ssh/config:  Host prod-*  User ec2-user  IdentityFile ~/.ssh/prod.pem  ProxyJump bastion
scp -i k.pem f user@host:/tmp/; sftp
/etc/ssh/sshd_config: PasswordAuthentication no; PermitRootLogin no; sshd -t; systemctl reload sshd
# AWS: aws ssm start-session --target i-xxx ; --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters host=db,portNumber=5432,localPortNumber=5432
```

## HTTP status codes quick ref

| Code | Meaning / usual DevOps cause |
|---|---|
| 301/302/307/308 | redirect (HTTP→HTTPS, trailing slash) |
| 400 | bad request, oversized headers |
| 401 / 403 | auth missing / forbidden (WAF, S3 policy, IAM) |
| 404 | wrong path, wrong target group, Ingress path mismatch |
| 408 | client timeout |
| 413 | body too large (nginx `client_max_body_size`, ALB 1 MB for Lambda) |
| 429 | rate limited / throttled |
| 460 (ALB) | client closed before idle timeout |
| 499 (nginx) | client closed connection |
| 500 | app exception |
| 502 | upstream returned garbage/closed: app crashed, wrong port, TLS mismatch, ALB target unhealthy |
| 503 | no healthy targets / service unavailable / K8s no endpoints |
| 504 | upstream timeout: ALB idle timeout 60 s, app slow, DB locked |

## OSI quick map

L2 MAC/ARP/switch · L3 IP/routing/ICMP · L4 TCP/UDP ports (NLB, SG) · L7 HTTP/TLS termination (ALB, Ingress, WAF).

TCP handshake SYN → SYN-ACK → ACK; `SYN_SENT` stuck = firewall/SG drop; `connection refused` = nothing listening (RST); `timeout` = packet dropped (SG/NACL/route).

## Subnetting

| CIDR | Hosts (AWS usable = −5) |
|---|---|
| /16 | 65,536 |
| /20 | 4,096 |
| /24 | 256 |
| /27 | 32 |
| /28 | 16 (AWS min) |

`ipcalc 10.0.0.0/20`; `python3 -c "import ipaddress;print(list(ipaddress.ip_network('10.0.0.0/22').subnets(new_prefix=24)))"`

## AWS-specific connectivity debug order

1. Security group inbound on target (port + source SG/CIDR)
2. Security group outbound on source
3. NACL inbound/outbound both subnets (ephemeral 1024–65535)
4. Route table of source subnet (IGW / NAT / TGW / peering / endpoint)
5. Is anything listening (`ss -tlnp` on target)? OS firewall?
6. DNS: private hosted zone / endpoint private DNS / Resolver rules
7. `aws ec2 describe-network-interfaces`; VPC Flow Logs (`REJECT`); Reachability Analyzer
8. ALB: target health reason (`aws elbv2 describe-target-health`), listener, health check path/port, deregistration
