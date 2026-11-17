# Pi-hole Installation and Initial Verification

## 1. Objective

The objective of this stage was to install Pi-hole on an Ubuntu virtual machine and verify that the DNS filtering service was operating correctly. Pi-hole acts as a DNS sinkhole by checking DNS requests against configured blocklists and returning blocking responses for domains that match those rules.

This document records the environment, installation, network configuration, and initial verification. Detailed interpretation of the DNS filtering experiments is provided separately in `Pihole-testing-analysis.md`.

**Note: All screenshots are located in level-1-foundation in the screenshots folder**

## 2. Environment

| Component | Configuration |
|---|---|
| Host operating system | Windows 11 |
| Virtualisation software | Oracle VirtualBox |
| Server operating system | Ubuntu 26.04.1 LTS |
| Server virtual machine | `DNS-Sinkhole-Server` |
| Client operating system | Ubuntu |
| Client virtual machine | `DNS-Sinkhole-Client` |
| DNS filtering software | Pi-hole |
| VirtualBox internal network | `dns-lab` |

The server and client each use two network adapters. Adapter 1 uses NAT for internet connectivity. Adapter 2 connects to the `dns-lab` internal network for communication within the isolated lab.

## 3. Pi-hole Installation Procedure

The Pi-hole installation script was downloaded from the official Pi-hole installation endpoint. The downloaded file was inspected before execution. Pi-hole was then installed by running the installation script and following the interactive configuration prompts.

The following configuration choices were made during installation:

- The private lab interface, `enp0s8`, was selected as the intended DNS interface.
- Google was selected as the upstream DNS provider.
- DNS query logging was enabled.
- The default privacy setting was retained.

The installation process completed successfully.

## 4. Server Network Configuration

The server has two IPv4 network interfaces.

| Interface | IPv4 address | Purpose |
|---|---|---|
| `enp0s3` | `10.0.2.15/24` | VirtualBox NAT connectivity |
| `enp0s8` | `192.168.50.10/24` | Private DNS lab network |

The private interface was configured with the static address `192.168.50.10/24`. The NAT interface remained available for software updates and upstream DNS resolution. The server's private address is used by the client when sending DNS requests to Pi-hole.

## 5. Pi-hole Service Verification

The command `sudo pihole status` was used to check the Pi-hole service. The output indicated that Pi-hole's FTL service was active, DNS was listening on port 53, and blocking was enabled.

The commands `ip -br addr` and `sudo ss -lntup | grep ':53'` were used to verify the network interfaces and inspect services listening on DNS port 53. The results confirmed that Pi-hole was listening for DNS requests. Other system services were also observed using port 5353, which is used for multicast DNS and is separate from the standard DNS port.

## 6. Web Dashboard Verification

The Pi-hole administration dashboard was accessed using the private lab address:

`http://192.168.50.10/admin/`

The dashboard loaded successfully and displayed the active Pi-hole status. At the initial observation, the dashboard showed approximately 74,481 domains on its lists. The query log was initially empty because client-generated DNS tests had not yet been performed.

## 7. Client VM Configuration

A separate Ubuntu client virtual machine was prepared to generate DNS requests for testing the Pi-hole server. The client was connected to the same VirtualBox internal network, `dns-lab`, as the server.

| Interface | IPv4 address | Purpose |
|---|---|---|
| `enp0s3` | NAT-assigned address | Internet connectivity |
| `enp0s8` | `192.168.50.20/24` | Private DNS lab network |

The client hostname was set to `DNS-Sinkhole-Client`.

| Device | Private lab IPv4 address |
|---|---|
| Pi-hole DNS server | `192.168.50.10` |
| Ubuntu client | `192.168.50.20` |

## 8. Client-to-Server Connectivity Verification

Connectivity was tested from the client using:

```bash
ping -c 4 192.168.50.10
```

The test returned four successful replies with zero packet loss. This confirmed that the two virtual machines could communicate over the private lab network.

## 9. Client DNS Configuration

The client was configured to use the Pi-hole server at `192.168.50.10` as its DNS resolver. NetworkManager was used to prevent the NAT connection from automatically replacing the configured DNS server:

```bash
sudo nmcli connection modify netplan-enp0s3 ipv4.ignore-auto-dns yes
sudo nmcli connection modify netplan-enp0s3 ipv4.dns "192.168.50.10"
sudo nmcli connection up netplan-enp0s3
```

The configuration was checked using:

```bash
resolvectl status
```

The output showed `192.168.50.10` as the DNS server for the client's NAT-connected interface. The internal lab interface had no default route.

## 10. Initial DNS Resolution Checks

DNS requests were sent directly to the Pi-hole server using `nslookup`.

The command below queried `example.com`:

```bash
nslookup example.com 192.168.50.10
```

The server returned IPv4 and IPv6 address records for `example.com`, demonstrating successful DNS forwarding and resolution before the domain was added to the denylist.

The Pi-hole local hostname was also checked:

```bash
nslookup pi.hole 192.168.50.10
```

The query resolved `pi.hole` to the Pi-hole server's IPv4 address, `192.168.50.10`. An IPv6 link-local address was also returned.

## 11. Query Log and Filtering Verification

The Pi-hole Query Log displayed DNS requests originating from client address `192.168.50.20`. It recorded requests for domains including `example.com`, `doubleclick.net`, `ubuntu.com`, and Ubuntu's connectivity-check domain.

The subsequent controlled allow/deny tests confirmed that permitted domains could be resolved and domains added to the exact-deny list returned sinkhole responses. The test procedure and interpretation are documented in `Pihole-testing-analysis.md`.

## 12. Evidence Collected

The following table maps the screenshot sequence supplied for this project to the corresponding evidence. The filenames below follow the numbered screenshot labels in the project screenshot collection. Keep the actual files in the `screenshots/` directory with these names.

| Screenshot | Filename | Evidence purpose |
|---:|---|---|
| 01 | `01-virtual-machine-created.png` | Initial server virtual machine setup. |
| 02 | `02-ubuntu-installation-and-network-check.png` | Ubuntu installation and initial network connectivity. |
| 03 | `03-ubuntu-package-updates.png` | Ubuntu package updates. |
| 04 | `04-pihole-installer-start.png` | Start of the Pi-hole installation process. |
| 05 | `05-static-ip-and-network-verification.png` | Server network interfaces, static private IP, and network verification. |
| 06 | `06-pihole-installation-complete.png` | Pi-hole installer completion. |
| 07 | `07-pihole-installation-confirmed.png` | Confirmation that Pi-hole installation completed. |
| 08 | `08-pihole-service-status-and-dns-port.png` | Pi-hole service status and DNS port verification. |
| 09 | `09-pihole-admin-dashboard-accessible.png` | Successful access to the Pi-hole administration dashboard. |
| 10 | `10-client-network-configuration.png` | Client virtual machine network configuration. |
| 11 | `11-client-hostname-and-ip-verification.png` | Client hostname and IP address verification. |
| 12 | `12-client-to-server-connectivity-test.png` | Connectivity test between the client and Pi-hole server. |
| 13 | `13-client-current-dns-configuration.png` | Client's current DNS resolver configuration. |
| 14 | `14-client-pihole-dns-server-configuration.png` | Configuration of the Pi-hole DNS server address on the client. |
| 15 | `15-client-networkmanager-connections.png` | Client NetworkManager connection settings. |
| 16 | `16-client-pihole-dns-configuration-confirmed.png` | Confirmation of the client's Pi-hole DNS configuration. |
| 17 | `17-client-dns-resolution-tests.png` | Initial DNS resolution tests from the client. |
| 18 | `18-pihole-query-log-client-dns-requests.png` | Pi-hole Query Log showing client DNS requests. |
| 19 | `19-allowed-dns-resolution-test.png` | Successful resolution of a permitted domain. |
| 20 | `20-pihole-domain-blocking-test.png` | Exact-deny rule for `example.com` and the blocking test. |
| 21 | `21-allowed-domain-resolution-test.png` | Successful resolution of a permitted domain after the deny rule was added. |
| 22 | `22-pihole-query-log-verification.png` | Query Log evidence for the allowed and blocked DNS requests. |
| 23 | `23-dns-cache-flush-blocking-verification.png` | Cache flush followed by a repeated blocked-domain lookup. |
| 24 | `24-multiple-domain-blocking-test.png` | Additional domain-blocking test. |
| 25 | `25-multiple-domain-query-log-verification.png` | Query Log verification for the additional domain tests. |
| 26 | `26-pihole-dashboard-query-activity.png` | Dashboard query-activity statistics. |
| 27 | `27-pihole-dashboard-domain-statistics.png` | Dashboard permitted- and blocked-domain statistics. |

## 13. Installation and Configuration Outcome

Pi-hole was installed successfully, its DNS service was active on port 53, and its administration dashboard was accessible through the private lab network. The client virtual machine was connected to the same internal network and configured to use the Pi-hole server at `192.168.50.10` for DNS resolution.

The initial connectivity and DNS checks succeeded. The subsequent filtering tests also demonstrated both permitted resolution and sinkhole responses for domains placed on the denylist. The detailed test results and limitations are analysed in the separate testing document.

## 14. Repository Status

The project documentation, screenshots, and test evidence are being maintained locally. The repository has not yet been uploaded to GitHub. Repository creation and publication will be completed after the project documentation and remaining validation are finalised.
