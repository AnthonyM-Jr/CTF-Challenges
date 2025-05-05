
# LetsDefend Walkthrough – Compromised AI Cluster

Welcome to this **question-by-question walkthrough** of the _Compromised AI Cluster_ challenge created by [LetsDefend](https://letsdefend.io)🛡️ In this scenario, we’re diving into a compromise involving an AI infrastructure running **Ray**—an open-source distributed computing framework for AI/ML workloads.

**Objective:** 
- Analyze a PCAP file, identify attacker behavior and extract indicators of compromise (IOCs)
- Answer 14 challenge questions to complete the challenge. 

---

## 🧰 Tools Used

- **Wireshark** – for network packet analysis
- **HTTP protocol knowledge** – to interpret requests/responses
- **Filters & Search** – to extract relevant activity

---

## 🔎 1. Identify the Active Software Framework

**📝 Question:** What software framework was active on the server at the time of the incident?

✅ **Answer:** `Ray`

![Q1](https://github.com/user-attachments/assets/df3befdf-c6b5-4a9f-8154-50de8febc4b9)
The software framework is identified in the initial summary. Additionally, its presence was confirmed in searching for the answer to question 2.

---

## 🧩 2. Determine the Framework Version

**📝 Question:** What version of the software framework was installed?

✅ **Answer:** `2.8.0`

🔍 Filter used:

```
Searched for the string "Version"
```
![Q2](https://github.com/user-attachments/assets/7392ab81-73d3-4fb7-a185-96c342cddac4)


Found version number in an HTTP response payload.

---

## 🌐 3. Public IP and Port of Ray Dashboard

**📝 Question:** What is the public IP address and port?

✅ **Answer:** `3.72.0.226:8265`

In HTTP request headers, the `Host:` field revealed the external IP and port where the Ray dashboard was publicly accessible.

![Q3](https://github.com/user-attachments/assets/16b01e34-4ee4-445e-bcde-8ea2dfbb8338)

---

## 👀 4. Persistent Malicious IP

**📝 Question:** Which IP persistently accessed the dashboard?

✅ **Answer:** `104.28.245.2`

![Q4](https://github.com/user-attachments/assets/43b9499c-8074-42cb-a62d-e5061679cb3a)



I filtered through the HTTP traffic and noticed an IP `(104.28.245.2)` repeatedly accessing:

- `/api/packages` 

- `/api/jobs`

- `/api/version`

Additionally, I noticed the use of a `/usr/bin/pwd` command in conjunction with the `/api/jobs` request, which lead me to believe the actor was conducting reconnaissance for exploitation:

---

## ⏱️ 5. First Attacker Contact

**📝 Question:** When did attacker first interact with the system?

✅ **Answer:** `2024-04-12 15:51:41 UTC`

![Q5](https://github.com/user-attachments/assets/a8b9f996-71bb-4a51-a1f8-ffb634613c38)

At this point in the investigation, I had noticed that three IP addresses shared the same first two octets `(104.28.x.x)`, suggesting they might be related or part of the same infrastructure. To explore this lead, I filtered the logs for activity from these three IPs. `This helped me identify the earliest interaction — an ICMP packet from 104.28.154.194 — which occurred on 2024-04-12 15:51:41 UTC.` Although this IP was different from the one I previously confirmed as persistently malicious, the shared prefix indicated a likely connection between them and helped trace the attacker’s initial footprint.

---

## 🧪 6. Initial Job Submission

**📝 Question:** What is the first job submission ID?

✅ **Answer:** `raysubmit_kkeJ8vLuM942ycqH`

Filter Used:

```
http.request.method == "POST" && http contains "/api/jobs"
```

![Q6](https://github.com/user-attachments/assets/680f9832-3834-458c-b08f-89b8f3deb821)

The first observed job submission resulting from attacker activity was associated with the submission ID `raysubmit_kkeJ8vLuM942ycqH`. This was triggered by a POST request from the attacker IP `104.28.213.2`, marking the point at which the adversary began interacting more directly with the system’s job handling functionality.

**Discussion on the preceding POST request in question 8!**

---

## 🐞 7. CVE Exploited

**📝 Question:** What vulnerability was used?

✅ **Answer:** `CVE-2023-48022`

![Q7](https://github.com/user-attachments/assets/860a2986-d530-43eb-a96b-01d43632c9d7)

During the process of reviewing all of the traffic from attacker IP 104.28.213.2, I discovered a malicious PUT request, containing a Python reverse shell script (reverseShell.py) embedded directly within the request body. The comments in the script made it clear that this was a proof-of-concept that had been weaponized to target CVE-2023-3676. This is a known vulnerability in Ray that allows remote attackers to execute arbitrary code by submitting crafted jobs to an exposed Ray Dashboard API. The exploit bypasses authentication and uses the job submission feature to execute the payload on the cluster node.


---

## 🖥️ 8. First Executed Command

**📝 Question:** What was the first command run?

✅ **Answer:** `whoami`

![Q8](https://github.com/user-attachments/assets/a9c0a38a-b1e5-488f-813b-72f9c59a74ed)

The attacker sent a POST request to the /api/jobs endpoint containing a whoami command. This command was likely used for reconnaissance to determine the username and privilege level of the account executing the remote code on the compromised system.

---

## 🛰️ 9. Reverse Shell Target

**📝 Question:** Where was the reverse shell initiated? What IP address and port were used?

✅ **Answer:** `52.150.25.174:1337`

![Q9](https://github.com/user-attachments/assets/8de0e4a0-3caa-4b24-ab10-ccb279577b28)

Following the whoami request, the image above shows a POST request sent to the server containing a reverse shell payload that initiates a reverse shell back to the C2 `52.150.25.174:1337`. Following the server's completion of this job, we see several packets of data between the server and the C2 that indicate that the reverse shell was successfully established.

![Q9-2](https://github.com/user-attachments/assets/37b3bfff-7d46-472c-b2ac-8cedf9994c4f)


---

## 📁 10. Reverse Shell Directory

**📝 Question:** From what directory was the reverse shell executed?

✅ **Answer:**  
`/tmp/ray/session_2024-04-12_11-13-55_403523_10331/runtime_resources/working_dir_files/_ray_pkg_bf19252c1fb036e5`

![image](https://github.com/user-attachments/assets/9bd18c24-762e-437f-8112-4b6ce51521a2)

By following the TCP stream between the AI cluster and malicious C2 server, I could see full directory that the reverse shell was executed from. Additionally, we can see that the attacker has root access, the highest level of privileges on the system. In the following image, we can see the attacker exploring the filesystem, successfully listing the contents of the `/home` directory using `ls /home`, and discovering a file named `secret.txt`. 

![image](https://github.com/user-attachments/assets/07f7b987-b4fe-44ca-973b-ea35243973df)


---

## 🛫 11. Secrets Exfiltration Job ID

**📝 Question:** What job was used to steal the secrets file?

✅ **Answer:** `raysubmit_7WZE36LAkVxbNc1x`

![Q14-3](https://github.com/user-attachments/assets/37a262d0-26c9-4343-9d0a-9c59dc7713aa)

Following the discovery of the `secret.txt` file, the attackers submitted a job to the server to obtain the contents of this file, receiving the job ID `raysubmit_7WZE36LAkVxbNc1x`. Following the job completion, the attackers receive the job logs containing the file contents which say `"Ops, you got me."(Answer to question 13)`

---

## 🔀 12. New IP Used in Later Stages

**📝 Question:** What new IP address was used?

✅ **Answer:** `104.28.213.2`

Referring back to the image on question 11, while the previous exploits were launched from a different IP, the request to fetch the job logs containing the `secret.txt` contents originated from a new attacker IP address: `104.28.213.2`.

---

## 📜 13. Contents of Exfiltrated File

**📝 Question:** What was in the exfiltrated file?

✅ **Answer:** `"Ops, you got me."`

---

## 🔓 14. Brute Force Behavior

**📝 Question:** The attacker deployed a tool to brute-force open ports on your network. How many packets were generated by this brute-force attempt?

Used Wireshark filter:

```
ip.src == 104.28.154.194 && tcp.flags.syn == 1 && tcp.flags.ack == 0 && ip.dst == 172.31.30.230
```

Navigated to:

```
Statistics > Endpoints > IPv4
```

✅ **Answer:** `39,598 SYN packets`

![Q14](https://github.com/user-attachments/assets/509a8223-68b5-4a69-bac7-19c2e10585f7)

Throughout the investigation I noticed a high number of SYN packets from `104.28.154.194`. Using the filters above I was able to pull the statistics and get the total number.

---

## 🧠 Lessons Learned

📌 **Never expose dev/admin dashboards to the public.**  
📌 Monitor job submissions and API abuse.  
📌 Use firewall rules to restrict access to sensitive ports.  

---

## 🚨 Indicators of Compromise (IOCs)

| Type           | Value                           |
|------------------|------------------------------------|
| Attacker IPs     | 104.28.245.2, 104.28.213.2, 104.28.154.194 |
| CVE Exploited    | CVE-2023-48022                    |
| Reverse Shell    | 52.150.25.174:1337                |
| Commands Run     | `whoami`, reverse shell script    |
| File Exfiltrated | `secret.txt`                      |

---

Thanks for reading!
