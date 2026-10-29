# Networking and troubleshooting cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: command → what it does and when you reach for it.

## Interfaces, routes, DNS config

- `ip a` / `ip -br a`

  Interfaces and addresses (brief form is one line each). Confirm the host has the IP you think it has.

- `ip link set eth0 up` / `ip addr add 10.0.0.5/24 dev eth0`

  Bring an interface up / add an address (temporary until reboot).

- `ip r`

  Routing table: which gateway/interface handles which destination.

- `ip route get 8.8.8.8`

  Which route and source IP a specific destination would use. Use when traffic leaves the wrong interface.

- `ip route add 10.1.0.0/16 via 10.0.0.1`

  Add a static route (temporary).

- `ip neigh`

  ARP cache: MAC addresses of neighbours. Stale entries after an IP move cause odd timeouts.

- `ss -tulpn`

  Listening TCP/UDP sockets with process (needs root for names). Answers "is anything listening on 8080". Replaces `netstat -tulpn`.

- `ss -tan state established`

  Established connections.

- `ss -s`

  Socket summary: counts by state. Thousands in TIME_WAIT or CLOSE_WAIT point at app connection handling.

- `ss -tnp dst :5432`

  Who is connected to the database port.

- `cat /etc/resolv.conf` / `resolvectl status`

  Which DNS servers the host uses (systemd-resolved stub 127.0.0.53 hides the real ones; `resolvectl` shows them).

- `/etc/hosts` / `/etc/nsswitch.conf`

  Static overrides / lookup order (files before dns). A stray hosts entry explains "dig is right but the app is wrong".

- `dig +short example.com`

  A record only.

- `dig example.com A @8.8.8.8`

  Ask a specific server: compare internal vs public answers for split-horizon issues.

- `dig -x 10.0.0.5`

  Reverse lookup.

- `dig example.com NS` / `CNAME +short` / `MX`

  Delegation / alias chain / mail. NS shows whether a zone is in Route 53 (`awsdns` servers).

- `dig +trace example.com`

  Walk the delegation from root servers. Use for "DNS works here but not there".

- `nslookup example.com` / `host example.com`

  Simpler lookups; `nslookup` exists on Windows too.

- `hostname -I` / `hostnamectl`

  This host's IPs / name.

## Connectivity tests

- `ping -c3 host`

  ICMP reachability and latency. Note: AWS security groups block ICMP unless allowed, so "no ping" ≠ "down".

- `ping -M do -s 1472 host`

  Don't-fragment ping with a 1500-byte frame. Fails = MTU problem on the path (VPNs, overlays).

- `traceroute -n host` / `mtr -rwc 20 host` / `tracepath host`

  Hop-by-hop path and loss; `mtr` runs continuously and shows per-hop loss. Use for "slow sometimes".

- `nc -zv host 443`

  TCP connect test to a port without sending data. The fastest "is the port open from here" check. `-u` for UDP.

- `nc -l 8080`

  Listen on a port to test inbound rules from another host.

- `timeout 3 bash -c '</dev/tcp/host/443' && echo open`

  Port test using bash's built-in /dev/tcp when `nc` isn't installed (minimal containers).

- `curl -v https://host`

  Verbose: shows DNS resolution, connect, TLS handshake, request and response headers. The all-in-one HTTP debugger.

- `curl -sI https://host`

  Headers only (HEAD). Check status and server headers quickly.

- `curl -o /dev/null -s -w '%{http_code} %{time_total}s\n' https://host`

  Just status code and total time. Script-friendly health check.

- `curl --resolve host:443:10.0.0.5 https://host/`

  Send the request to a specific IP while keeping the Host/SNI. Tests one backend behind a load balancer or a DNS change before it propagates.

- `curl -k` / `curl --cacert ca.pem`

  Skip TLS verification (diagnosis only) / trust a private CA.

- `curl -H 'Host: x' http://ip/`

  Override the Host header: test virtual hosts/Ingress rules by IP.

- `curl -x http://proxy:3128 https://x`

  Go through a proxy; also honours `https_proxy` env var.

- `curl -X POST -H 'Content-Type: application/json' -d '{"a":1}' https://x/api`

  Send a JSON POST.

- `curl -w '@fmt.txt' ...`

  Custom timing breakdown (dns, connect, tls, ttfb) from a format file. Pinpoints which phase is slow.

- `wget -qO- url` / `wget --spider url`

  Fetch to stdout / check existence without downloading.

- `telnet host 25`

  Interactive raw TCP (SMTP, Redis).

## TLS and certificates

- `openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null | openssl x509 -noout -dates -subject -issuer`

  Fetch the served certificate and print validity dates, subject, issuer. `-servername` sets SNI so you get the right cert on shared IPs. The expiry check.

- `openssl s_client -connect host:443 -showcerts </dev/null`

  Full chain as served. Missing intermediate = "unable to get local issuer certificate" on clients.

- `openssl x509 -in cert.pem -noout -text`

  Decode a certificate file: SANs, key usage, validity.

- `openssl x509 -in cert.der -inform der -out cert.pem`

  Convert DER → PEM.

- `openssl req -new -newkey rsa:2048 -nodes -keyout k.pem -out r.csr -subj '/CN=x'`

  Generate a key and CSR for a CA (or ACM import).

- `openssl rsa -in k.pem -check`

  Validate a private key.

- `openssl verify -CAfile ca.pem cert.pem`

  Check a cert chains to a CA.

- `openssl x509 -noout -modulus -in cert.pem | md5sum` vs `openssl rsa -noout -modulus -in k.pem | md5sum`

  Same hash = this key matches this cert. Use when a load balancer rejects a cert/key pair.

- `echo | openssl s_client -connect host:443 2>/dev/null | grep -E 'Protocol|Cipher'`

  Negotiated TLS version and cipher. Compliance checks (TLS 1.2+).

- `update-ca-trust` (RHEL, drop CA in `/etc/pki/ca-trust/source/anchors/`) / `update-ca-certificates` (Debian, `/usr/local/share/ca-certificates/`)

  Install a corporate/TLS-inspection CA so `curl`, `pip`, `aws` trust it. Fixes "certificate verify failed" behind proxies.

## Packet capture

- `tcpdump -i any -nn port 443`

  Live capture on all interfaces, numeric (no DNS/port lookups). See whether packets arrive at all.

- `tcpdump -i eth0 -nn host 10.0.0.5 and port 8080`

  Filter by peer and port.

- `tcpdump -i any -nn -w cap.pcap -c 1000` / `tcpdump -r cap.pcap`

  Save 1000 packets to a file for Wireshark / read it back.

- `tcpdump -i any -nn 'tcp[tcpflags] & (tcp-syn) != 0 and not src net 10.0.0.0/8'`

  Only SYNs from outside the private range: who's knocking.

- `tcpdump -i any -nn -A port 80 | grep Host`

  Print payload as ASCII; see Host headers on plain HTTP.

- `tshark -i eth0 -Y 'http.request'`

  Wireshark CLI with display filters.

Reading a capture: SYN with no SYN-ACK = dropped (firewall/SG/route or host down). SYN then RST = reached host, port closed. Retransmissions = loss or MTU.

## Host firewall and SELinux

- `iptables -L -n -v` / `iptables -t nat -L -n -v` / `iptables -S`

  List filter rules with counters / NAT rules / rules as commands. Counters show whether a rule is matching.

- `iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT` then `iptables -P INPUT DROP`

  Allow SSH from the corporate range, then default-deny inbound. Order matters; persist with `iptables-save`.

- `nft list ruleset`

  nftables (the modern backend) rules.

- `firewall-cmd --list-all` / `--add-port=8080/tcp --permanent` / `--reload`

  firewalld (RHEL): view / open a port persistently / apply.

- `ufw status` / `ufw allow 443/tcp` / `ufw enable`

  Ubuntu's simple firewall.

- `getenforce` / `setenforce 0`

  SELinux mode / set permissive temporarily to test whether SELinux is the blocker (then fix the policy, don't leave it off).

- `ausearch -m avc -ts recent`

  Recent SELinux denials with the process and target.

- `semanage port -l | grep http` / `semanage port -a -t http_port_t -p tcp 8080`

  Ports a confined service may bind / allow a new one.

- `setsebool -P httpd_can_network_connect 1`

  Allow a confined web server to make outbound connections (reverse proxies).

## SSH

- `ssh -i key.pem -J bastion user@private-host`

  Jump through a bastion in one command (ProxyJump).

- `ssh -L 8080:internal:80 user@bastion`

  Local port forward: `localhost:8080` → `internal:80` via the bastion. Reach a private dashboard or DB.

- `ssh -R 9000:localhost:3000 user@host`

  Remote forward: expose your local 3000 on the remote host's 9000. Webhook testing.

- `ssh -D 1080 user@host`

  SOCKS proxy through the host; point a browser at it to browse as if inside the VPC.

- `ssh -vvv user@host`

  Verbose handshake: which key was offered, why auth failed.

- `ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5`

  Non-interactive options for scripts (accept risk of no host verification).

- `ssh-keygen -t ed25519 -C 'me'` / `ssh-copy-id user@host`

  Generate a modern key / install it on a host.

- `ssh-add -l`

  Keys loaded in the agent.

- `ssh-keygen -R host`

  Remove a stale known_hosts entry after a host was rebuilt ("REMOTE HOST IDENTIFICATION HAS CHANGED").

- `~/.ssh/config`: `Host prod-*  User ec2-user  IdentityFile ~/.ssh/prod.pem  ProxyJump bastion`

  Per-host defaults so `ssh prod-web1` just works.

- `scp -i k.pem f user@host:/tmp/` / `sftp`

  Copy files over SSH. `rsync -e ssh` for big trees.

- `/etc/ssh/sshd_config`: `PasswordAuthentication no`, `PermitRootLogin no`; `sshd -t`; `systemctl reload sshd`

  Hardening, test config syntax, apply without dropping sessions.

- `aws ssm start-session --target i-xxx`

  Shell on an instance with no SSH port open, logged in CloudTrail. The bank-preferred way.

- `aws ssm start-session --target i-xxx --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters host=db,portNumber=5432,localPortNumber=5432`

  Port-forward to a private RDS through an instance, no bastion SSH.

## HTTP status codes a DevOps engineer meets

- **301/302/307/308**

  Redirect → HTTP→HTTPS rules, trailing slash, wrong scheme header behind a proxy

- **400**

  Bad request → oversized headers/cookies, malformed Host

- **401 / 403**

  Unauthenticated / forbidden → WAF block, S3 bucket policy, IAM, OIDC auth on ALB, Akamai/CDN edge

- **404**

  Not found → wrong path rule, wrong target group, Ingress path mismatch, app not deployed

- **408**

  Client took too long to send

- **413**

  Payload too large → nginx `client_max_body_size`, ALB→Lambda 1 MB limit

- **429**

  Rate limited → WAF rate rule, API throttling, app limiter

- **460 (ALB)**

  Client closed before ALB idle timeout → client timeout shorter than backend latency

- **499 (nginx)**

  Client closed connection

- **500**

  Application exception → app logs

- **502**

  Bad gateway → upstream crashed, wrong port, TLS mismatch to backend, target unhealthy, response too large/invalid

- **503**

  No healthy targets / overloaded → health checks failing, zero ready pods (empty Endpoints), Kubernetes Service with no backends

- **504**

  Gateway timeout → ALB idle timeout (60 s default) < app response time, DB lock, slow upstream

## Concepts in one line

- L2 MAC/ARP/switch · L3 IP/routing/ICMP · L4 TCP/UDP ports (NLB, security groups) · L7 HTTP/TLS termination (ALB, Ingress, WAF).
- TCP handshake SYN → SYN-ACK → ACK. Stuck `SYN_SENT` = drop; `connection refused` = RST, nothing listening; `timeout` = dropped by SG/NACL/route.
- Stateful (SG) vs stateless (NACL): NACLs need return-traffic rules on ephemeral ports 1024–65535.

## Subnetting

| CIDR | Addresses (AWS usable = minus 5) | Typical use |
|---|---|---|
| /16 | 65,536 | One VPC |
| /20 | 4,096 | Large subnet (EKS nodes/pods) |
| /24 | 256 | Standard subnet |
| /27 | 32 | Small tier (NAT, endpoints) |
| /28 | 16 | AWS minimum subnet |

`ipcalc 10.0.0.0/20` for boundaries; `python3 -c "import ipaddress;print(list(ipaddress.ip_network('10.0.0.0/22').subnets(new_prefix=24)))"` to split.

## AWS "can't connect" debug order

1. Security group **inbound** on the target: port and source (CIDR or SG).
2. Security group **outbound** on the source (default allows all; custom ones often don't).
3. NACLs on both subnets, both directions, including ephemeral ports.
4. Route table of the source subnet: IGW / NAT / TGW / peering / endpoint route present?
5. Is anything listening on the target (`ss -tlnp`)? Host firewall?
6. DNS: does the name resolve to what you expect (private hosted zone, endpoint Private DNS, Resolver rules)?
7. Evidence: VPC Flow Logs (`REJECT` lines name the blocking side), `aws ec2 describe-network-interfaces`, Reachability Analyzer for a definitive path verdict.
8. Load balancer: `aws elbv2 describe-target-health` reason codes, listener rules, health-check path/port, deregistration delay.
