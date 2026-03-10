# FMC MCP Server — Prompt Library

## Tool Domain Map

The FMC MCP server exposes tools across the **Cisco Secure Firewall Management Center** REST API (`fmc_config`, `fmc_platform`, and `fmc_tid` namespaces). It supports **multi-FMC profile management** — a single server instance can front multiple FMC deployments simultaneously: [cisco](https://www.cisco.com/c/en/us/td/docs/security/secure-firewall/management-center/API/REST_API_config_guide/secure-firewall-management-center-rest-api-for-access-control-policy.html)

| Domain | Core Tools |
|---|---|
| **FMC Profiles** | `list_fmc_profiles`, `select_fmc_profile` |
| **Device Management** | `get_ftd_devices`, `get_device_detail`, `get_ha_pairs`, `get_device_clusters`, `get_device_health` |
| **Access Control** | `get_access_policies`, `get_access_rules`, `find_rules_by_ip_or_fqdn`, `find_rules_for_target`, `search_access_rules` |
| **Network Objects** | `get_network_objects`, `get_host_objects`, `get_fqdn_objects`, `get_network_groups`, `create_network_object` |
| **Port / Service Objects** | `get_port_objects`, `get_port_groups`, `get_protocol_port_objects` |
| **Intrusion Policies** | `get_intrusion_policies`, `get_intrusion_rules`, `get_snort_rules` |
| **Security Intelligence** | `get_si_feeds`, `get_si_lists`, `get_url_categories`, `get_dns_policies` |
| **NAT Policies** | `get_nat_policies`, `get_nat_rules`, `get_auto_nat_rules` |
| **VPN** | `get_s2s_vpn_topologies`, `get_ra_vpn_profiles`, `get_vpn_ike_settings` |
| **File / Malware Policies** | `get_file_policies`, `get_malware_cloud_lookup`, `get_amp_settings` |
| **Platform Settings** | `get_platform_settings_policies`, `get_syslog_servers`, `get_dns_servers` |
| **Deployment** | `get_pending_changes`, `deploy_to_devices`, `get_deployment_jobs` |
| **Audit & Health** | `get_audit_logs`, `get_health_alerts`, `get_health_monitor_status` |
| **URL Filtering** | `get_url_objects`, `get_url_categories_reputation`, `get_url_filtering_policies` |
| **Prefilter** | `get_prefilter_policies`, `get_prefilter_rules`, `get_tunnel_tags` |

> ⚠️ **Multi-FMC support:** Always begin prompts by specifying the FMC profile (e.g., `fmc-north-south`, `fmc-datacenter`) to ensure the correct FMC instance is targeted. [developer.cisco](https://developer.cisco.com/codeexchange/github/repo/CiscoDevNet/CiscoFMC-MCP-server-community/)

***

## 🔥 Category 1 — Firewall Device Management

For **Security Engineers and NOC Teams** managing Cisco FTD device fleets.

1. **"List all FTD devices managed by the `fmc-datacenter` FMC profile. Show device name, model, software version, registration status, associated access policy, and last heartbeat time. Flag any device that has not synced in the last 15 minutes."**

2. **"Show me all HA (High Availability) pairs configured across all FMC profiles. For each pair, show the active/standby state, last failover time, and whether both units are currently healthy and in sync."**

3. **"Are there any FTD devices that have pending configuration changes not yet deployed? List the device name, associated FMC profile, what changed (policy name, rule count delta), and how long the change has been pending."**

4. **"Get the health status for all FTD devices on the `fmc-branch` FMC profile. Flag any device reporting CPU above 80%, memory above 85%, or disk usage above 75%."**

5. **"Which FTD devices are running a software version older than 7.4? Show device name, current version, FMC profile, and the recommended upgrade path based on Cisco's recommended release track."**

6. **"Show all FTD cluster members in the `DC-Cluster-East` device cluster. What is the current role of each member (control/data), their load distribution, and is the cluster fully operational with all members active?"**

***

## 📋 Category 2 — Access Control Policy Audit

For **Security Architects, Compliance Teams, and Firewall Admins** auditing access control policy hygiene.

7. **"List all access control policies across all FMC profiles. For each policy, show the name, assigned devices, number of rules, default action (block/trust/allow), and last modified date."**

8. **"Show me all access rules in the `HQ-North-South-Policy` that have an action of `ALLOW` with NO intrusion policy and NO file policy attached. These rules are passing traffic without any inspection — they represent a security gap."**

9. **"Find all access rules in the `Branch-Perimeter-Policy` that use the `any` source network or `any` destination network AND have an action of `ALLOW`. Rank them by rule position — overly broad allow rules near the top of the list are the highest risk."**

10. **"Which access rules across all policies have never matched any traffic (hit count = 0) in the last 90 days? List them with policy name, rule name, position, and creation date — these are candidates for cleanup."**

11. **"Show all access rules that are currently disabled across all policies on the `fmc-datacenter` FMC. List the policy, rule name, rule action, and when it was last active. Disabled rules with sensitive actions should be reviewed for removal."**

12. **"Audit all access rules where the `Log at End of Connection` or `Log at Beginning of Connection` setting is NOT enabled. Unlogged rules create blind spots in our security posture — list them by policy and rule position."**

***

## 🔍 Category 3 — Rule Search & Traffic Investigation

For **Security Operations, Incident Response, and Change Management** teams performing policy lookups and impact analysis.

13. **"Find all access rules in the `Perimeter-FW-Policy` that match traffic from source IP `203.0.113.55`. Show every rule it would hit — in order — with the action (Allow/Block/Trust) and whether intrusion inspection is enabled."**

14. **"Find all access rules in any policy that permit traffic to destination FQDN `malware-c2.badactor.com`. This FQDN has been flagged as a threat IOC — I need to know if any rule is explicitly allowing it through."**

15. **"Use `find_rules_for_target` to resolve device `FTD-EDGE-CHICAGO` to its assigned access policies, then search for any rule permitting inbound TCP port 3389 (RDP) from external zones. Show all matching rules with their source/destination scope."**

16. **"Search for all access rules across all FMC profiles that reference Security Group Tag (SGT) `PCI-Servers` as a source or destination condition. Show the full rule detail — policy, position, action, and any associated intrusion/file policy."**

17. **"I need to check the impact of blocking IP `185.220.101.47`. Use `search_access_rules` to find all rules that currently PERMIT traffic from this IP across all policies. Show the rule details so I can assess what would break if we blocked it."**

18. **"Find all access rules that reference the network object group `CORP-RFC1918-SUBNETS` as a destination. I'm planning to modify that object group and need the full blast radius — which rules and policies would be affected?"**

***

## 🧱 Category 4 — Network & Service Object Management

For **Firewall Admins and NetDevOps Teams** managing the object library.

19. **"List all network objects and network groups currently defined in FMC on the `fmc-datacenter` profile. Show the object name, type (host/network/range/FQDN), value, and which access rules reference it. Flag any objects that are not referenced by any rule — these are orphaned objects."**

20. **"Are there any duplicate network objects in FMC? Find all cases where two or more objects have the same IP address or CIDR range but different names. Duplicates cause policy confusion and should be consolidated."**

21. **"Show all FQDN objects defined in FMC. Which ones have FQDNs that are no longer resolving to valid IP addresses? These stale FQDN objects may cause unintended policy behavior."**

22. **"List all port objects and port groups. Identify any port group that contains TCP port 22, 23, 3389, or 445 in a group named something 'generic' — these high-risk ports should be in explicitly named objects, not bundled into broad service groups."**

23. **"I need to create a new network host object named `PROD-API-SERVER-01` with IP `10.50.100.25` on the `fmc-datacenter` FMC profile. Show me the object definition before creating it. After creation, confirm it appears in the object list."**

***

## 🛡️ Category 5 — Intrusion Prevention (IPS) & Snort

For **Security Engineers and SOC Analysts** managing threat detection and IPS tuning.

24. **"List all intrusion policies configured in FMC. For each policy, show the name, base policy it inherits from, number of custom rule overrides, and which access rules (and devices) are currently assigned to it."**

25. **"Which intrusion policy is applied to the rule permitting traffic from the `INTERNET-ZONE` to the `DMZ-WEBSERVERS` network group in the `Perimeter-FW-Policy`? Show the base policy name and whether it uses the `Balanced Security and Connectivity` or `Security over Connectivity` base — and is it deployed to all relevant devices?"**

26. **"Are there any Snort 3 intrusion rules that have been manually set to `Disabled` in our production policies? List the rule GID:SID, rule message, policy name, and who disabled it and when — via the audit log."**

27. **"Show all intrusion events from the last 4 hours where the impact flag is `Impact 1` (vulnerable) or `Impact 2` (potentially vulnerable). Group by rule name, source IP, and destination. Flag any events where the destination is in the `PCI-Servers` network group."**

***

## 🌐 Category 6 — Security Intelligence & URL Filtering

For **Security Engineers and Threat Intelligence Teams** managing feed-based blocking.

28. **"Show all Security Intelligence (SI) feeds currently configured in FMC — IP reputation, URL, and DNS feeds. For each feed, show the provider, update frequency, last successful update, and which access policies are consuming it."**

29. **"Is Security Intelligence blocking enabled in the `Perimeter-FW-Policy`? Show the SI objects applied to the ingress and egress SI lists — and confirm the `Block List` is set to `Block` and not just `Monitor`."**

30. **"List all custom Security Intelligence IP block lists and URL block lists manually maintained in FMC. Show the entries in each list, who last modified them, and the date of last update. Flag any list that hasn't been updated in more than 30 days."**

31. **"Show the URL filtering categories and reputations configured in FMC. Which categories are set to `Block` vs `Allow with Warning`? Flag if `Malware Sites`, `Phishing` or `Command-and-Control` categories are NOT set to block — these are critical threat categories."**

***

## 🔒 Category 7 — NAT & VPN

For **Network Security Engineers** managing address translation and secure connectivity.

32. **"List all NAT policies in FMC and their assigned devices. For the `HQ-Edge-NAT-Policy`, show all static and dynamic NAT rules, the original source/destination, translated address, and interface zones."**

33. **"Show all site-to-site VPN topologies configured in FMC. For each topology, show the tunnel name, hub and spoke endpoints, IKE version, encryption algorithm, and whether the VPN is currently up or down."**

34. **"Are there any S2S VPN tunnels using IKEv1 with DES or 3DES encryption? These are cryptographically weak configurations that should be migrated to IKEv2 with AES-256. List affected topologies and their assigned FTD devices."**

35. **"Show the Remote Access VPN (RA VPN) profile configuration on the `fmc-datacenter` FMC. What authentication method is configured (certificate, RADIUS, SAML), which FTD devices are the VPN headends, and how many concurrent sessions are allowed?"**

***

## 📁 Category 8 — File Policy & AMP/Malware Defense

For **SOC and Endpoint Security Teams** managing file inspection and malware protection.

36. **"List all file policies defined in FMC. For each policy, show the name, file types being inspected, actions (Detect, Block, Malware Cloud Lookup, Block Malware), and which access rules have this file policy assigned."**

37. **"Are there any access rules in production policies that permit traffic without a file policy applied, where the destination is in the `INTERNAL-SERVERS` zone? These are servers receiving unscanned file transfers — show them with rule position and traffic volume if available."**

***

## 🚀 Category 9 — Deployment & Change Management

For **Change Advisory Boards and Firewall Admins** managing controlled deployments.

38. **"What configuration changes are currently pending deployment across all FTD devices on the `fmc-datacenter` FMC profile? Show each device, the pending change summary (which policy changed, how many rules added/modified/deleted), and the admin who made the change."**

39. **"Show the deployment history for the `FTD-EDGE-HQ` device for the last 7 days. List each deployment job — timestamp, deploying admin, success/failure status, and the policies that were deployed."**

40. **"The pending changes on `FTD-EDGE-NYC` include a new block rule for IP range `198.51.100.0/24`. Before deploying, confirm: (1) Is this IP range already referenced in any existing allow rule? (2) Does any current active session from this range exist in the connection table? Show me the impact analysis, then confirm before deploying."**

***

## 🏭 Vertical-Specific Prompt Packs

### Financial Services / PCI-DSS
- *"Audit all access policies assigned to FTD devices in the `PCI-Zone` device group. List every rule that permits ANY traffic into the cardholder data environment — show source, destination, service, and whether IPS and file inspection are both enabled on each permitting rule."*

### Healthcare / HIPAA
- *"Show all access rules permitting traffic from external zones to the `ePHI-Servers` network group. For each rule, confirm that both an intrusion policy and a file policy with malware blocking are applied. Flag any rule where either is missing."*

### Government / Zero Trust
- *"List all access rules in the `Classified-Segment-Policy` that have an action other than BLOCK. For each allow rule, show the full condition set — source SGT, destination, user/group condition, time-range constraint, and inspection policy. Any allow rule without all four condition types should be flagged for review."*

### Manufacturing / OT Security
- *"List all access rules in the `OT-DMZ-Policy` that permit traffic originating from the `IT-Zone` destined for the `OT-Zone`. These east-west IT-to-OT flows are the highest-risk paths in an industrial network — show rule detail, hit counts, and whether ICS protocol inspection (Modbus, DNP3) is enabled."*

### MSP / Multi-FMC Management
- *"List all FMC profiles configured in the server. For each FMC, show how many FTD devices it manages, how many access policies exist, total rule count, and whether there are any pending deployments. Give me a cross-FMC operational health snapshot."*

***

## 🔁 Cross-Ecosystem / Multi-MCP Prompts

The FMC MCP server is the **perimeter and segmentation enforcement layer** of the suite. It answers the "what is the firewall doing about it?" question that complements every other server's data. [github](https://github.com/CiscoDevNet/CiscoFMC-MCP-server-community)

***

### 🔗 FMC + ISE — Policy Intent vs. Enforcement Reality

41. **"ISE assigned SGT `Contractors` to endpoint `a4:c3:f0:9b:12:44`. Use FMC's `search_access_rules` with identity indicator `SGT=Contractors` to show every access rule across all policies that applies to that SGT. Confirm the contractor is being correctly blocked from reaching the `CORP-SERVERS` zone as intended by the segmentation policy."**

42. **"ISE has just quarantined a compromised endpoint and changed its SGT to `Quarantine`. Pull all FMC access rules that reference the `Quarantine` SGT and confirm a BLOCK rule is present in all perimeter and internal segmentation policies — verify enforcement is end-to-end."**

***

### 🔗 FMC + Splunk — Firewall Event Correlation

43. **"Splunk has flagged high-volume traffic from internal IP `10.10.50.77` to external destination `185.220.101.0/24`. Use FMC to find which access rule is permitting that traffic — show the rule action, associated intrusion policy, and whether any IPS events have been raised for this flow. Then search Splunk's `index=firepower_events` for connection and intrusion events involving the same source IP in the last 2 hours."**

44. **"Search Splunk's `index=firepower_events` for all `Impact Flag 1` intrusion events in the last 24 hours. For the top 5 triggered rules, use FMC to pull the full Snort rule text and the access rule context — which access policy, what zone pair, and what is the destination asset? Build a prioritized threat brief."**

***

### 🔗 FMC + ThousandEyes — Connectivity and Policy Triage

45. **"ThousandEyes is showing packet loss from the Singapore enterprise agent to our data center application. Use FMC to check the access policy on `FTD-DC-EDGE` — is there a block rule that could be causing drops? Run `find_rules_by_ip_or_fqdn` for the Singapore agent's egress IP against the inbound access policy."**

***

### 🔗 FMC + Catalyst Center + ISE — End-to-End Zero Trust Verification

46. **"A new user segment has been deployed. Verify end-to-end Zero Trust enforcement: (1) ISE — confirm the user group maps to the correct SGT and authorization profile; (2) Catalyst Center — confirm the SGT is propagated via TrustSec to the correct switch VLAN and port; (3) FMC — confirm the SGT-based access rules in the perimeter and data center firewall policies enforce the correct ALLOW/DENY matrix. Report any gap between policy intent and deployed enforcement."**

***

### 🔗 FMC + IOS-XE + Splunk — Multi-Layer Block Verification

47. **"We need to block all traffic from IP range `198.51.100.0/24` immediately after a threat intelligence hit. Orchestrate across three MCP servers: (1) FMC — deploy a new block rule to all perimeter FTD devices; (2) IOS-XE — push an ACL to the WAN edge routers to block at the routing layer as defense-in-depth; (3) Splunk — create a saved search alert to notify if any traffic from that range bypasses both layers and reaches internal systems. Show all proposed changes before executing any."**

***

## Prompt Engineering Tips for FMC MCP

| Principle | Guidance |
|---|---|
| **Always specify FMC profile first** | Multi-FMC deployments require `fmc_profile=<profile-name>` scoping on every tool call — omitting it may target the wrong FMC  [developer.cisco](https://developer.cisco.com/codeexchange/github/repo/CiscoDevNet/CiscoFMC-MCP-server-community/) |
| **Use profile aliases** | FMC profiles support aliases (`north`, `north-south`, `10.0.0.5`) — use the most human-readable alias in prompts for clarity  [developer.cisco](https://developer.cisco.com/codeexchange/github/repo/CiscoDevNet/CiscoFMC-MCP-server-community/) |
| **Rule search before change** | Always run `find_rules_by_ip_or_fqdn` or `search_access_rules` before adding new rules — duplicate or conflicting rules are a common FMC issue  [developer.cisco](https://developer.cisco.com/codeexchange/github/repo/CiscoDevNet/CiscoFMC-MCP-server-community/) |
| **Verify hit counts** | When auditing for rule cleanup, always anchor to hit count data — a zero-hit rule may simply be a backup rule, not an unused one |
| **Deployment confirmation gate** | Any prompt involving `deploy_to_devices` must include "show pending changes and list target devices — confirm before deploying"  [cisco](https://www.cisco.com/c/en/us/td/docs/security/secure-firewall/management-center/API/REST_API_config_guide/secure-firewall-management-center-rest-api-for-access-control-policy.html) |
| **SGT as policy anchor** | For TrustSec environments, always use SGT as the search indicator in `search_access_rules` — it provides the most precise cross-domain policy match  [developer.cisco](https://developer.cisco.com/codeexchange/github/repo/CiscoDevNet/CiscoFMC-MCP-server-community/) |
| **Audit log for accountability** | Use `get_audit_logs` after any security incident to establish a chain of custody — who changed what rule, when, and from which admin session |

***

The FMC MCP server is the **security enforcement oracle** of the MCP-Suite. It uniquely answers not just "what policy is configured?" but "what will the firewall actually do with this traffic?" — making it indispensable for Zero Trust verification, incident response, compliance auditing, and pre-change impact analysis across the entire SDN Tech Forum suite. [github](https://github.com/CiscoDevNet/CiscoFMC-MCP-server-community)
