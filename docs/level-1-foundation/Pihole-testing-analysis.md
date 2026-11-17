# Pi-hole DNS Sinkhole Testing and Analysis

## 1. Introduction

This document analyses the functional tests carried out on the Pi-hole DNS sinkhole lab. The tests were designed to establish whether a separate client virtual machine could send DNS requests through Pi-hole, whether permitted domains could resolve normally, and whether domains placed on Pi-hole's denylist would receive sinkhole responses.

The testing environment consisted of an Ubuntu Pi-hole server and an Ubuntu client running in Oracle VirtualBox. Both virtual machines were connected to the private `dns-lab` network.

## 2. Test Environment

| Component | Configuration |
|---|---|
| DNS filtering server | `DNS-Sinkhole-Server` |
| Server private IP | `192.168.50.10` |
| Client | `DNS-Sinkhole-Client` |
| Client private IP | `192.168.50.20` |
| DNS software | Pi-hole |
| DNS port | UDP/TCP 53 |
| Upstream DNS provider | Google, as selected during installation |
| Virtual network | VirtualBox internal network `dns-lab` |

DNS queries were sent explicitly to the Pi-hole server using `nslookup <domain> 192.168.50.10`. This made the resolver under test clear and avoided relying only on the client's automatically selected resolver.

## 3. Testing Method

The tests were conducted in stages:

1. Verify that the client could communicate with the Pi-hole server.
2. Confirm that the client was configured to use Pi-hole for DNS.
3. Query a permitted domain and record the returned DNS addresses.
4. Add a controlled domain to Pi-hole's exact-deny list.
5. Flush the client DNS cache and repeat the query.
6. Confirm the blocked result in the Pi-hole Query Log.
7. Repeat the process with another domain and inspect the dashboard statistics.

The client-side `nslookup` output was compared with the Pi-hole Query Log and dashboard. Screenshots are referenced by their numbered filenames in the `screenshots/` directory.

## 4. Results

### 4.1 Client-to-server connectivity

The client was able to reach the Pi-hole server at `192.168.50.10`. The recorded ping test returned four replies with zero packet loss. This established IP connectivity across the private lab network and allowed the DNS tests to proceed.

**Evidence:** `12-client-to-server-connectivity-test.png`

### 4.2 Client DNS configuration

The client DNS configuration was checked with `resolvectl status`. The output showed the Pi-hole server address, `192.168.50.10`, as the configured DNS resolver. The network configuration screenshots also document the NetworkManager settings used to prevent the NAT connection from replacing the selected DNS server.

**Evidence:** `13-client-current-dns-configuration.png`, `14-client-pihole-dns-server-configuration.png`, `15-client-networkmanager-connections.png`, and `16-client-pihole-dns-configuration-confirmed.png`

This configuration, together with the explicit server address in the `nslookup` commands, supports the conclusion that the test queries were directed to Pi-hole.

### 4.3 Permitted domain resolution

The initial query for `example.com` returned public IPv4 addresses (`172.66.147.243` and `104.20.23.154`) and IPv6 addresses. The query was sent to `192.168.50.10`, and the Pi-hole Query Log showed the corresponding A and AAAA requests from client `192.168.50.20`.

The Pi-hole local hostname was also tested. The query for `pi.hole` returned the server's private IPv4 address, `192.168.50.10`.

A later query for `ubuntu.com` returned public IPv4 and IPv6 addresses, showing that a domain not added to the denylist continued to resolve through Pi-hole.

**Evidence:** `17-client-dns-resolution-tests.png`, `18-pihole-query-log-client-dns-requests.png`, `19-allowed-dns-resolution-test.png`, and `21-allowed-domain-resolution-test.png`

**Analysis:** These results demonstrate that Pi-hole was not simply rejecting all DNS traffic. It successfully forwarded permitted requests and returned usable DNS records to the client.

### 4.4 Blocking `example.com`

The domain `example.com` was added to Pi-hole as an exact-deny entry. After the rule was enabled, the client DNS cache was flushed using:

```bash
sudo resolvectl flush-caches
```

The client then repeated the query:

```bash
nslookup example.com 192.168.50.10
```

The response contained `0.0.0.0` for the IPv4 lookup and `::` for the IPv6 lookup, rather than the public addresses returned during the earlier permitted test. This is consistent with Pi-hole's sinkhole response for a blocked domain.

The Query Log displayed the `example.com` A and AAAA requests as blocked. The dashboard's Top Blocked Domains panel later showed four hits for `example.com`.

**Evidence:** `20-pihole-domain-blocking-test.png`, `22-pihole-query-log-verification.png`, and `23-dns-cache-flush-blocking-verification.png`

**Analysis:** The before-and-after comparison is important. The same domain resolved to public addresses before the deny rule was applied and returned sinkhole addresses after the rule was enabled and the client cache was flushed. This supports the conclusion that the deny rule was effective during the test.

### 4.5 Blocking `doubleclick.net`

A second domain, `doubleclick.net`, was queried through the Pi-hole server. The client received `0.0.0.0` and `::` as the returned addresses. The corresponding A and AAAA requests appeared in the Pi-hole Query Log with blocked status.

The dashboard's Top Blocked Domains panel showed two hits for `doubleclick.net`.

**Evidence:** `24-multiple-domain-blocking-test.png` and `25-multiple-domain-query-log-verification.png`

**Analysis:** The second test indicates that the filtering behaviour was not limited to the first manually added test domain. Pi-hole also blocked `doubleclick.net` according to the active filtering rules. The evidence available here confirms the DNS-level blocking response; it does not by itself establish that every application or browser connection to the domain was prevented by other network controls.

### 4.6 Dashboard statistics and query activity

The Pi-hole dashboard displayed query activity from the client and separated permitted and blocked domains. The Top Blocked Domains panel listed `example.com` with four hits and `doubleclick.net` with two hits. The Top Permitted Domains panel included domains such as `api.beacondb.net`, `connectivity-check.ubuntu.com`, `cdn.fwupd.org`, and `github.com`.

**Evidence:** `26-pihole-dashboard-query-activity.png` and `27-pihole-dashboard-domain-statistics.png`

**Analysis:** The dashboard provides a useful summary of the DNS activity observed during the testing period. It corroborates the individual `nslookup` and Query Log results by showing both permitted and blocked activity. The totals are specific to the captured test session and should not be interpreted as long-term traffic measurements or a benchmark of Pi-hole performance.

## 5. Consolidated Test Summary

| Test | Expected behaviour | Observed result | Assessment |
|---|---|---|---|
| Client ping to `192.168.50.10` | Replies from the Pi-hole server | Four replies, zero packet loss | Passed |
| Client DNS configuration | Pi-hole address used as resolver | `192.168.50.10` shown in resolver configuration | Passed |
| `example.com` before deny rule | Public DNS records returned | Public IPv4 and IPv6 records returned | Passed |
| `pi.hole` lookup | Pi-hole local hostname resolves | Resolved to `192.168.50.10` | Passed |
| `ubuntu.com` lookup | Permitted domain resolves | Public IPv4 and IPv6 records returned | Passed |
| `example.com` after exact-deny rule | Sinkhole response returned | `0.0.0.0` and `::` returned after cache flush | Passed |
| `doubleclick.net` lookup | Blocked domain receives sinkhole response | `0.0.0.0` and `::` returned | Passed |
| Query Log verification | Client requests and filtering decisions recorded | Client `192.168.50.20` and allowed/blocked entries visible | Passed |
| Dashboard statistics | Summary reflects DNS activity | Permitted and blocked domain statistics displayed | Passed |

## 6. Discussion

The results show that the lab achieved its primary functional objective: a separate client machine sent DNS requests to Pi-hole, and Pi-hole handled permitted and denied domains differently. Permitted domains such as `example.com` during the initial baseline test and `ubuntu.com` returned public DNS records. After `example.com` was added to the exact-deny list, the same lookup returned sinkhole addresses. The second blocked-domain test for `doubleclick.net` produced the same type of sinkhole response.

Using multiple forms of evidence strengthened the test. The client terminal showed the DNS responses, while the Pi-hole Query Log showed that the requests came from the expected client address and recorded their filtering status. The dashboard then provided a higher-level summary of permitted and blocked query activity.

The cache-flush step was also relevant. DNS responses can be cached by the client, so repeating a lookup without clearing the cache could potentially show a previously cached result rather than the current filtering decision. Flushing the resolver cache before repeating the `example.com` query reduced this source of ambiguity.

## 7. Limitations

The tests demonstrate DNS-level filtering in a small, controlled virtual environment. They do not establish that all forms of advertising, tracking, or malicious activity can be blocked. DNS filtering only affects requests that use the Pi-hole resolver and domains represented by active filtering rules.

The tests were performed using a limited set of domains and a single client virtual machine. They were functional checks rather than a large-scale performance or security evaluation. No quantitative comparison was made against another DNS filtering product, and the dashboard query totals represent only the observed lab session.

The `nslookup` results establish the DNS responses returned by Pi-hole. They do not prove that a browser, application, or operating system would never contact a related service through a hard-coded IP address, another resolver, encrypted DNS, or an alternative domain.

## 8. Conclusion

The testing successfully demonstrated the core operation of the Pi-hole DNS sinkhole. The client communicated with the server, used `192.168.50.10` for DNS resolution, and generated requests visible in the Pi-hole Query Log. Permitted domains resolved to public DNS addresses, while domains configured for blocking returned sinkhole responses (`0.0.0.0` and `::`).

The combination of client-side `nslookup` output, cache-flush verification, Query Log entries, and dashboard statistics provides consistent evidence that Pi-hole's DNS filtering was functioning as intended in the controlled lab environment.

## 9. Evidence Index

| Screenshot | Filename | Main use in this analysis |
|---:|---|---|
| 12 | `12-client-to-server-connectivity-test.png` | Confirms client/server IP connectivity. |
| 13-16 | `13-client-current-dns-configuration.png` to `16-client-pihole-dns-configuration-confirmed.png` | Documents client DNS configuration. |
| 17 | `17-client-dns-resolution-tests.png` | Initial client DNS lookup tests. |
| 18 | `18-pihole-query-log-client-dns-requests.png` | Shows client requests in the Query Log. |
| 19 | `19-allowed-dns-resolution-test.png` | Baseline permitted lookup. |
| 20 | `20-pihole-domain-blocking-test.png` | Shows the deny rule and blocked-domain setup. |
| 21 | `21-allowed-domain-resolution-test.png` | Shows a permitted lookup. |
| 22 | `22-pihole-query-log-verification.png` | Confirms filtering status in the Query Log. |
| 23 | `23-dns-cache-flush-blocking-verification.png` | Shows cache flush and repeat blocked lookup. |
| 24-25 | `24-multiple-domain-blocking-test.png` and `25-multiple-domain-query-log-verification.png` | Additional blocked-domain test and log evidence. |
| 26-27 | `26-pihole-dashboard-query-activity.png` and `27-pihole-dashboard-domain-statistics.png` | Dashboard-level summary of DNS activity. |
