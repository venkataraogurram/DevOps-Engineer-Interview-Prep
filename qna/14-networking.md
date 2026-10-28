# 14. Networking

[← Back to index](README.md)

**Q115. Walk through what happens when a user hits https://app.bank.com behind an ALB on EKS.**
DNS: Route 53 alias resolves to ALB IPs (one per AZ). TCP handshake to the ALB, TLS handshake with the ACM certificate (SNI selects it). The ALB evaluates listener rules by host and path, picks a target group; with the AWS Load Balancer Controller in IP mode the targets are pod IPs. The ALB opens a new connection to the pod through the node's ENI (VPC CNI gives pods VPC IPs), subject to the node or pod security group. The pod's readiness determines whether it's registered. Response returns the same path. WAF, if attached, evaluates before routing. Access logs land in S3.

---

**Q116. `connection refused` vs `connection timed out`. What does each tell you?**
Refused means the packet reached the host and the kernel replied with RST: nothing is listening on that port, or a host firewall rejects with RST. So network path and security groups are fine; check the process and port. Timed out means no reply at all: packets dropped by a security group, NACL, route, or host firewall DROP rule, or the host is down. On AWS, security groups drop silently, so a timeout almost always means SG, NACL or routing.

---

**Q117. How do security groups differ from NACLs and when does a NACL bite you?**
Security groups are stateful at the ENI; return traffic is automatically allowed, allow-only, can reference other SGs. NACLs are stateless at the subnet, ordered allow/deny rules, so you must allow the ephemeral port range 1024–65535 for return traffic in the opposite direction. A NACL bites you when someone adds a deny rule with a lower number than your allow, or when a new subnet pair gets a custom NACL and the ephemeral return rule is missing: inbound works, replies die.

---

**Q118. Explain NAT gateway vs internet gateway vs VPC endpoint.**
Internet gateway: gives public-subnet resources with public IPs a route to the internet, both directions. NAT gateway: lets private-subnet resources initiate outbound to the internet without inbound exposure; one per AZ for HA; charges per GB processed. VPC endpoints: private paths to AWS services; gateway endpoints for S3 and DynamoDB are free and route-table based; interface endpoints (PrivateLink) create ENIs with private IPs and can replace NAT for AWS API traffic, which saves money and keeps traffic off the internet, which banks want.

---

**Q119. Interface endpoint with Private DNS enabled broke something. What could it be?**
Private DNS makes the service's public hostname resolve to the endpoint's private IPs inside the VPC. If the endpoint's security group doesn't allow 443 from the callers, or the endpoint policy denies the action, or a service has a console or sub-hostname that the endpoint doesn't serve, calls that used to go out via NAT now fail. Also hybrid DNS: on-prem clients resolving through your Route 53 Resolver inbound endpoint get private IPs they can't reach. Diagnose with `dig` for the hostname inside the VPC and `aws ec2 describe-vpc-endpoints` for the SG and policy.

---

**Q120. VPC peering vs Transit Gateway vs PrivateLink?**
Peering: one-to-one, non-transitive, no overlapping CIDRs, free within a region except data transfer; fine for a handful of VPCs. Transit Gateway: hub-and-spoke, transitive routing, route tables per attachment for segmentation, connects VPN and Direct Connect too; the standard for tens of VPCs in a bank. PrivateLink: expose one service, not a network; consumer gets an ENI in their VPC pointing at your NLB; overlapping CIDRs are fine; use for SaaS-style or cross-team service exposure with no network trust.

---

**Q121. How does TLS termination work on ALB, and when do you re-encrypt to the target?**
ALB terminates TLS using an ACM certificate and can present multiple certs via SNI. Backend traffic can be HTTP or HTTPS; for HTTPS targets the ALB does not validate the target certificate, so self-signed is fine. Banks often require encryption in transit end to end, so re-encrypt to the pods or instances. For strict end-to-end with client certs or your own CA validation, use NLB with TLS passthrough and terminate at the application or a mesh with mTLS.

---

**Q122. Explain DNS TTL and how it affects a blue/green cutover.**
TTL is how long resolvers cache a record. Before a cutover, lower the TTL (e.g. 60 s) at least one old-TTL period in advance so caches expire quickly; switch the record; raise TTL afterwards. Clients and corporate resolvers may ignore TTLs, and Java apps cache DNS unless configured, so a load balancer switch (listener rule or weighted target groups) is more deterministic than a DNS switch. Route 53 alias records to ALB have no customer-set TTL; the ALB's own IPs change, so never hardcode them.

---

**Q123. What is MTU and how does it cause weird hangs?**
Maximum transmission unit, 1500 bytes on most paths, 9001 jumbo inside a VPC. If a path segment has a smaller MTU and ICMP "fragmentation needed" is blocked, large packets silently drop: small requests work, big responses hang. Classic over VPNs and some overlay networks. Test with `ping -M do -s 1472 host`; fix by lowering MTU on the interface or enabling MSS clamping. Transit Gateway supports 8500 bytes, VPN 1500, so hosts talking across a VPN with jumbo frames need clamping.

---

**Q124. How do you expose an internal service on EKS to another AWS account privately?**
Service type LoadBalancer with the AWS LB Controller creating an internal NLB, then a VPC Endpoint Service (PrivateLink) on that NLB, allowlisting the consumer account's principal. The consumer creates an interface endpoint and gets a private DNS name. No peering, no CIDR coordination, one-directional, and the consumer's security group controls access on their side.

---

**Q125. Explain how a pod reaches the internet on EKS and how you'd restrict it.**
The pod has a VPC IP from the node's ENI in a private subnet; traffic routes through the subnet's NAT gateway. To restrict: NetworkPolicy default-deny egress with allowlists per namespace (VPC CNI network policy or Cilium), security groups for pods for AWS-side control, VPC endpoints for AWS APIs, and an egress proxy or AWS Network Firewall for domain-based filtering, which banks usually require for outbound internet.

---

**Q126. What is the difference between L4 and L7 load balancing, with AWS examples?**
L4 routes TCP/UDP connections by IP and port without reading content: NLB, kube-proxy, fast, preserves source IP, TLS passthrough possible. L7 understands HTTP: ALB, Ingress controllers, API Gateway; routes by host, path, header, can authenticate, rewrite, do WAF, and distributes per request rather than per connection, which matters for gRPC and HTTP/2 multiplexed connections.

---

**Q127. How does Route 53 health checking and failover work, and what are the limits?**
Health checkers from multiple regions probe an endpoint; the record is healthy if a majority agree. Failover routing returns the secondary when the primary is unhealthy. Limits: checkers must reach the endpoint from the internet unless you use CloudWatch-alarm-based health checks for private resources; DNS caching delays the switch by the TTL; health checks don't understand your application's semantics beyond a status code or string match. Latency, weighted, and geolocation routing can each be combined with health checks.

[← Back to index](README.md)
