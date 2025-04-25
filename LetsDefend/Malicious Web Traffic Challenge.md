## Malicious Web Traffic Analysis Challenge
**Posted by Anthony | Cybersecurity Analyst | April 2025**
## 🛠️ Tools Used

![Wireshark](https://img.shields.io/badge/Wireshark-005CAD?style=for-the-badge&logo=wireshark&logoColor=white)
![Cyber Chef](https://img.shields.io/badge/Cyber%20Chef-FF4136?style=for-the-badge&logo=apache&logoColor=white)
## 🧠 Introduction
In this post, I’ll walk you through a Malicious Web Traffic Analysis challenge I completed recently on LetsDefend. The scenario was based on a real-world investigation into suspicious network behavior involving a web server. What followed was a fascinating chain of exploitation—from an XML External Entity attack to brute-force access, file enumeration, and even an open redirect.

Let’s break down the attack chain step by step and uncover what the adversary did and how they did it.

## 🖥️ Phase 1: Finding the Web Server and Attacker
Upon initial analysis of the PCAP file provided for the investigation, I identified two IP addresses that stood out:
- **Web Server IP**: 10.1.0.4 - I observed a number of GET, POST requests to and from this server, identifying it as our internal Web server.
- **Attacker IP**: 197.32.212.121 – This external IP sent multiple ACK packets to port 3389 on the web server, which aligns with a potential ACK port scan. This immediately stood out as suspicious activity.


![Phase 1](https://github.com/user-attachments/assets/f6bf8453-80be-4671-8663-38603afc8790)

## 📦 Phase 2: Exploiting an XXE Vulnerability
Looking deeper, I noticed multiple HTTP POST requests to `/register/register.php`. One request carried a particularly suspicious XML payload:
![Phase 2](https://github.com/user-attachments/assets/9b8cc7e0-0312-4816-b39c-5b8007089894)

Through research, this is a textbook XML External Entity (XXE) attack. The attacker tried to extract the source code of register.php by encoding it in base64 and injecting the result into the response.


## 📦 Phase 3: Hidden Note in the Source Code
In the web server’s response to the attempted XXE exploit, there is a base64 encoded note in the source code:
| Encoded String |
|----------------|
| `PD9waHAKbGlieG1sX2Rpc2FibGVfZW50aXR5X2xvYWRlciAoZmFsc2UpOwokeG1sZmlsZSA9IGZpbGVfZ2V0X2NvbnRlbnRzKCdwaHA6Ly9pbnB1dCcpOwokZG9tID0gbmV3IERPTURvY3VtZW50KCk7CiRkb20tPmxvYWRYTUwoJHhtbGZpbGUsIExJQlhNTF9OT0VOVCB8IExJQlhNTF9EVERMT0FEKTsKJGluZm8gPSBzaW1wbGV4bWxfaW1wb3J0X2RvbSgkZG9tKTsKJG5hbWUgPSAkaW5mby0+bmFtZTsKJHRlbCA9ICRpbmZvLT50ZWw7CiRlbWFpbCA9ICRpbmZvLT5lbWFpbDsKJHBhc3N3b3JkID0gJGluZm8tPnBhc3N3b3JkOwojI0FkbWluLCB0aGlzIGNvbW1lbnQgaXMganVzdCB0byBsZXQgeW91IGtub3cgdGhhdCB3ZSB1cGRhdGVkIHlvdXIgY3JlZGVudGlhbHMgd2l0aCBhIHZlcnkgc2VjdXJlIHBhc3N3b3JkLiBzbyBub29uZSBjYW4gYnJ1dGUgZm9yY2UgaXQuCiMjTm90ZTogc3VibWl0IG1lIGFzIHRoZSBhbnN3ZXI6IHlvdWdvdG1lCmVjaG8gIlNvcnJ5LCAkZW1haWwgaXMgYWxyZWFkeSByZWdpc3RlcmVkISI7Cj8+Cg==` |

Once decoded using Cyber Chef, the base64 string revealed a PHP script with a hidden note:
| Decoded Note |
|----------------|
| ##Admin, this comment is just to let you know that we updated your credentials with a very secure password.
##Note: submit me as the answer: yougotme` |


This was not just a hint for the CTF—it also revealed a probable admin username: Admin.

## 🔐 Phase 4: Brute-Forcing the Admin Account
With **Admin** in hand, the attacker initiated a brute-force login attack and eventually succeeded using:

- **Username**: admin  
- **Password**: fernando
![Phase 4](https://github.com/user-attachments/assets/f7b14203-668d-4bcc-afab-095ca19580ca)

This gave them unauthorized access to the admin panel. At this point, the attacker had a foothold.

## 📂 Phase 5: Directory Traversal to View System Files
The attacker’s next move was to probe the server’s file system using a classic directory traversal payload:
| Traversal Payload |
|----------------|
| ../../../../../../../../../../../../../../../etc/passwd |

![Phase 5](https://github.com/user-attachments/assets/b1b56d6e-1a79-41f7-a6aa-75ef6d78a4ee)

This allowed them to read the contents of `/etc/passwd`, which listed all user accounts on the server.

## 👤 Phase 6: Identifying New Users
Among the listed users, the last created user was `a1l4mFTW`—a unique string likely generated during the system’s configuration or by another automated script.
![Phase 6](https://github.com/user-attachments/assets/1972d630-f4e0-46d1-bd9c-f29d3861c662)

## 🔁 Phase 7: Open Redirect Vulnerability
To wrap it up, the attacker uncovered one final issue: an **open redirect** vulnerability.
![Phase 7-1](https://github.com/user-attachments/assets/0dce919c-a923-4c43-94e4-a660fb16f6aa)

They tested it by sending users to a malicious domain using a crafted URL:
| Malicious Domain |
|----------------|
| `GET /?url=https://evil[.]com/` |
![Phase 7-2](https://github.com/user-attachments/assets/faa9ff30-332a-492d-8866-a4aa149b9411)


An open redirect like this can be exploited in phishing campaigns, where users are tricked into thinking they’re visiting a legitimate site, only to be redirected elsewhere.

## 🔚 Conclusion
This CTF was a powerful reminder of the cascading effect a single vulnerability can have if left unchecked. Here's a recap of the key points:

| **Attack Phase**         | **Detail**                          |
|--------------------------|--------------------------------------|
| Web Server IP            | 10.1.0.4                            |
| Attacker IP              | 197.32.212.121                      |
| Initial Exploit          | XXE via `register.php`             |
| Hidden Answer            | `yougotme`                         |
| Admin Login              | `admin:fernando`                   |
| Directory Traversal      | Read `/etc/passwd`                 |
| Last Created User        | `a1l4mFTW`                         |
| Open Redirect Test       | `https://evil[.]com/`              |

This type of hands-on analysis is what makes cybersecurity both challenging and rewarding. If you're diving into CTFs, I highly recommend dissecting traffic like this—it sharpens your skills and gives real-world context to theoretical vulnerabilities.

## 💬 Got questions or thoughts about this challenge?
Drop a comment or reach out—I’d love to talk shop.
