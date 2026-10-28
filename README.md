# Tomcat Takeover Wireshark Lab


## Project Overview
This project uses network traffic data from the Tomcat Takeover lab on CyberDefenders. The objective was to analyze suspicious network traffic, reconstruct the attack sequence, identify evidence of unauthorized access, and investigate how the attacker achieved remote command execution on the victim web server.

- **Tools:** Wireshark, VirusTotal
- **Evidence source:** Network traffic capture (PCAP)

## Skills Demonstrated
- Network traffic analysis with Wireshark
- TCP stream inspection
- Web application attack investigation
- Reverse shell identififaction
- Security findings documentation

## Investigation Findings Summary
##### **Attacker IP:** 14.0.0.120
##### **Victim IP:** 10.0.0.112
##### **Target port:** TCP/8080

- **Reconnaissance:** Multiple port-scanning attempts targeted the victim host to identify open ports and accessible services.
- **Web enumeration:** The attacker made requests to web paths and interfaces on the Tomcat server, consistent with directory enumeration.
- **Credential attack:** The attacker made repeated authentication attempts against the Tomcat Manager interface before successfully gaining access.
- **Application deployment:** A WAR file named `JXQOZY.war` was uploaded to the server.
- **Remote command execution:** Subsequent network traffic is consistent with a reverse shell connecting back to the attacker.
- **Persistence:** Evidence indicates that a cron job was configured to attempt recurring reverse-shell connections.

#### Find the full investigation report [here](https://github.com/sbaneesa/Tomcat-Takeover-Wireshark-Lab/blob/main/Investigation.md)

## Security Vulnerabilities and Misconfiguirations
The attack was facilitated by an exposed Apache Tomcat service on TCP port 8080, which provided the attacker with an entry point for web application enumeration. Although an open port is not inherently a vulnerability, exposing the service to unauthorized users increases the server's attack surface.

Through directory enumeration, the attacker identified web resources and located the Tomcat management interface. The interface was protected by weak credentials, which the attacker successfully compromised through repeated authentication attempts. This allowed the attacker to access administrative functionality and deploy a malicious WAR file to execute commands on the server and establish a reverse shell.




## Recommended Remediation
- **Detect port scanning:** Configure network monitoring and intrusion detection rules to identify repeated connection attempts across multiple ports.
- **Restrict administrative access:** Limit access to the Tomcat Manager interface to authorized hosts or management networks. Disable the interface if it is not required.
- **Strengthen authentication:** Replace default or weak credentials with strong, unique credentials and follow applicable organizational password policies and NIST guidance.
- **Monitor authentication activity:** Alert on repeated failed login attempts, unusual authentication patterns, and successful logins following multiple failures.
- **Secure application deployment:** Restrict WAR deployment permissions to authorized administrators and review deployed applications for unauthorized changes.
- **Detect persistence and command execution:** Monitor for suspicious cron jobs, unexpected outbound connections, and processes that may indicate reverse-shell activity.
