# Pi-hole DNS Sinkhole Security Lab

## Project Overview

This project demonstrates the deployment and testing of a DNS sinkhole using Pi-hole in a controlled VirtualBox lab environment.

The objective was to understand how DNS filtering works, configure a dedicated DNS server, and investigate how malicious or unwanted domains can be blocked at the DNS level.

## Lab Architecture

| Component | Configuration |
|---|---|
| DNS Server | Ubuntu Server with Pi-hole |
| DNS Server IP | `192.168.50.10` |
| Client Machine | Ubuntu |
| Client IP | `192.168.50.20` |
| Virtualisation | Oracle VirtualBox |
| DNS Protocol | DNS over port 53 |

## Tools and Technologies

- Ubuntu Linux
- Pi-hole
- Oracle VirtualBox
- DNS and `nslookup`
- Linux networking tools
- Git and GitHub

## Testing Performed

The following tests were conducted:

1. DNS server connectivity verification.
2. DNS resolution for permitted domains.
3. Blocking of `example.com`.
4. Blocking of `doubleclick.net`.
5. Verification of A and AAAA DNS responses.
6. DNS cache flushing and repeat testing.
7. Pi-hole query log verification.
8. Dashboard statistics and domain activity analysis.

## Key Findings

- The client successfully used Pi-hole as its DNS server.
- Permitted domains resolved successfully.
- Blocked domains returned sinkhole responses (`0.0.0.0` and `::`).
- Pi-hole recorded DNS requests from the client in its query log.
- Dashboard statistics provided visibility into permitted and blocked DNS activity.

## Project Documentation

- [Pi-hole Installation Guide](docs/Pihole-installation-updated.md)
- [Testing and Analysis](docs/Pihole-testing-analysis.md)
- [Project Log](docs/project-log.txt)

## Evidence

Screenshots documenting the installation, configuration, DNS testing, query logs, and dashboard results are available in the `screenshots/` directory.

## Security Note

This is an isolated educational lab. The DNS server is intended for controlled testing and is not exposed as a public DNS resolver.

## Future Improvements

- Investigate DNS traffic using Wireshark.
- Analyse Pi-hole query logs.
- Develop a Python script to summarise DNS activity.
- Investigate DNS filtering limitations and bypass techniques.