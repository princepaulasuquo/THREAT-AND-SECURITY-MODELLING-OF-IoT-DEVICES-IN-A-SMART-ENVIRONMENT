# Threat and Security Modelling of IoT Devices in a Smart Environment

**MSc Information Technology with Cybersecurity – final project, Robert Gordon University (2025)**
**Author:** Prince Paul Asuquo

This project assesses and hardens a simulated smart home. It combines STRIDE threat modelling with practical testing in Nmap, Wireshark and RouterSploit. Every test was run twice, once on the default configuration and once after hardening, so the effect of each security control can be measured. The raw scan and capture outputs from both phases are in this repository.

---

## Problem

Smart home IoT devices can introduce network vulnerabilities through weak authentication, poor update mechanisms and flat network design. Organisations have security policies and central monitoring; home users rarely do, and most lack the expertise to find or fix these issues. This project tests a practical, repeatable way to assess and improve a smart home's security using a virtual testbed and open-source tools, with no specialist hardware needed.

## Aim and objectives

**Aim:** evaluate and enhance the security of IoT devices in a smart environment by combining structured threat modelling with practical hardening.

1. Apply STRIDE threat modelling to a simulated smart home system.
2. Carry out reconnaissance and vulnerability analysis with Nmap, Wireshark and RouterSploit.
3. Test how well the hardening measures work: HTTPS, IP banning and credential enforcement.

## Test environment

| Component | Details |
|---|---|
| Smart home hub | Home Assistant OS running in an Oracle VirtualBox virtual machine |
| Simulated devices | Smart lights, IP camera feed, door locks, thermostat and sensors, integrated through Home Assistant |
| Exposed service | Home Assistant web interface on TCP 8123 |
| Networking | VirtualBox NAT with host port 8123 forwarded to the VM |
| Test platform | Windows 10 host running Nmap, Wireshark and RouterSploit against `127.0.0.1:8123` |

The baseline used Home Assistant's default web settings: plain HTTP on port 8123, a single password-protected administrator account, no TLS and no network access restrictions. This reflects a typical home installation where the user changes little after setup.

## Methodology

1. **Threat model.** Draw a data flow diagram (user, Home Assistant hub, IoT devices, external services) and identify threats in each STRIDE category.
2. **Baseline assessment.** Scan, capture and attack the default configuration.
3. **Hardening.** Apply controls chosen from the threat model.
4. **Re-assessment.** Repeat the same tests with the same tools and compare the results with the baseline.

### STRIDE summary

| Category | Threat in this smart home | How it was tested | Mitigation applied |
|---|---|---|---|
| Spoofing | Attacker impersonates the user or a device; password-only login on a flat network | RouterSploit default-credential checks | Strong, unique admin password; IP ban after 5 failed logins; MFA recommended |
| Tampering | Commands or sensor readings altered in transit over HTTP | Wireshark | TLS on the web interface |
| Repudiation | Malicious actions not recorded | Review of Home Assistant logs | IP-ban logging creates an audit trail of failed logins |
| Information disclosure | Web traffic readable on the network; mDNS broadcasts advertising device names | Wireshark | TLS; no debug interfaces or public endpoints |
| Denial of service | Flooding port 8123 or repeated login attempts | Assessed in the threat model (not load-tested) | Login threshold; interface not exposed to the internet |
| Elevation of privilege | Software flaws allowing privileged access | RouterSploit exploit modules; Nmap port review | Fully updated software; strong admin credentials; minimal install from official sources |

## Tools and settings

**Nmap:** full-port scan with service and version detection.

```
nmap -p- -sV 127.0.0.1
```

**Wireshark:** captured on the host's loopback interface while loading the Home Assistant login page and signing in. Display filters were used to isolate web interface traffic and to look for other device traffic such as mDNS.

**RouterSploit:** the `scanners/autopwn` module, plus `creds` modules for Telnet/SSH default credentials.

| Setting | Baseline | After hardening |
|---|---|---|
| Target | 127.0.0.1 | 127.0.0.1 |
| HTTP port | 8123 | 8123 |
| `http_ssl` | false | true |

## Hardening applied

- **HTTPS (SSL/TLS)** enabled on the web interface by adding a certificate and key to the `http:` section of `configuration.yaml`.
- **IP banning:** an IP address is banned and logged after five failed logins.
- **Strong credentials:** a high-strength administrator password, with no guessable demo accounts.
- **Service minimisation:** confirmed with Nmap that no unneeded ports were listening.
- **Updates:** Home Assistant core and the host OS updated before testing.

Configuration used (certificate paths shown as placeholders):

```yaml
http:
  ssl_certificate: /ssl/<certificate>.pem
  ssl_key: /ssl/<private-key>.pem
  ip_ban_enabled: true
  login_attempts_threshold: 5
```

## Results: before and after

| Check | Baseline | After hardening |
|---|---|---|
| Nmap open ports | One: 8123/tcp, identified as HTTP (Python aiohttp) | Still only 8123/tcp, now shown as `polipo?` because the port expects a TLS handshake; no new ports opened |
| Wireshark | HTTP GET requests and 200 OK responses readable in plain text; some mDNS packets advertising device names | TLSv1.2 handshake followed by encrypted Application Data on port 8123 |
| RouterSploit autopwn | No exploit succeeded; the scan stopped at the Heartbleed module because of a dependency error | Same result: no exploit succeeded; same module error |
| RouterSploit creds | Telnet/SSH default-credential attempts failed or found no service | Not applicable: no additional services exposed |
| Brute-force protection | None | IP banned after 5 failed logins, confirmed in the Home Assistant logs |
| Browser view | `http://`, marked "Not secure" | `https://` with padlock; login required |

**Summary:** the main baseline weakness was the unencrypted management interface. Anyone on the network could read its traffic, and a submitted login would have crossed the network in clear text. After hardening, traffic is encrypted, brute-force attempts are blocked and logged, and the attack surface stayed at a single port.

## Repository contents

| File | Contents |
|---|---|
| `nmap_baseline.txt` | Nmap output before hardening |
| `nmap_after.txt` | Nmap output after hardening |
| `rsf_before.txt` | RouterSploit output before hardening (`http_ssl` false) |
| `rsf_after.txt` | RouterSploit output after hardening (`http_ssl` true) |
| `wireshark_before.pcapng` | Packet capture before hardening (plain-text HTTP) |
| `wireshark_after.pcapng` | Packet capture after hardening (TLS) |

Tip: open the captures in Wireshark and filter on `tcp.port == 8123` to compare the two phases.

## Limitations

- Virtual devices only. No physical hardware, firmware or radio protocols (Zigbee, Z-Wave, Bluetooth LE) were tested.
- The scope was network-layer and configuration hardening. No intrusion detection system was deployed.
- RouterSploit's autopwn scan did not complete in either phase because of a dependency error. "No findings" therefore does not prove the system is secure; manual testing or another framework would be needed for full assurance.
- Nmap's default service scan misidentified the TLS-protected service. TLS-aware version probing is needed for accurate fingerprinting.
- Tests ran from the host against `127.0.0.1` through NAT port forwarding. Bridged networking would better reflect a real home network with several hosts.
- IP banning could be abused to lock out legitimate users. Ban logs should be monitored and trusted networks allow-listed.

## Lessons learned

- **Tool output needs interpretation.** The `polipo?` label after hardening was a side effect of TLS, not a new service.
- **Encryption trades visibility for security.** Once traffic was encrypted, packet captures showed metadata only, so logs became the main source of evidence.
- **Correlate tools with logs.** Home Assistant's logs confirmed that the IP ban worked after deliberate failed logins.
- **Match tool settings to the target.** A wrong `http_ssl` setting at first caused connection failures in RouterSploit.

## Recommendations

- Encrypt all IoT traffic with TLS 1.2 or 1.3 and keep certificates current.
- Minimise open ports and services, and place IoT devices on a separate network segment (VLAN) behind a firewall.
- Replace default passwords with strong, unique ones; enable MFA, rate limiting and account lockout.
- Patch hubs and devices promptly from trusted sources.
- Monitor IoT traffic with IDS/IPS or SIEM tooling, and include IoT devices in incident response plans.
- Prefer devices that meet ETSI EN 303 645 and follow OWASP IoT Top 10 guidance.

## Future work

- Physical devices and other protocols (Zigbee, Z-Wave, Bluetooth LE), plus cloud-connected services such as Home Assistant Cloud.
- Anomaly-based intrusion detection that works on encrypted IoT traffic.
- Larger deployments with many devices, users and network segments.
- Privacy-focused threat modelling frameworks such as LINDDUN and PASTA.

## Ethics

All testing took place in a lab environment that the author owned and controlled. No scanning or exploitation was directed at any system without permission, in line with the UK Computer Misuse Act 1990.

## Acknowledgements

Thanks to Dr Shahana Bano and the RGU Cybersecurity Lab for guidance and resources.

## Key references

- Antonakakis, M. et al. (2017). *Understanding the Mirai Botnet.* 26th USENIX Security Symposium. https://www.usenix.org/system/files/conference/usenixsecurity17/sec17-antonakakis.pdf
- Fagan, M. et al. (2020). *Foundational Cybersecurity Activities for IoT Device Manufacturers* (NISTIR 8259). https://doi.org/10.6028/nist.ir.8259
- Home Assistant documentation. https://www.home-assistant.io/

The full dissertation is available on request.
