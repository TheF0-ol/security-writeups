---
title: "IDOR in chat transcript download endpoint"
slug: "port-swigger-access-control-idor-chat-transcript"
authors: ["Shravan Kakkar (TheF0-ol)"]
date: "2026-10-14"
tags: ["Active Directory", "VAPT", "Privilege Escalation", "Enumeration"]
summary: "Solved the PortSwigger Access Control / IDOR Apprentice lab. Chat transcripts were served via predictable filenames with no ownership check. Recovered a plaintext password and logged in as the victim account."
---

## 1. Overview

- Objective: Solve the PortSwigger *Insecure direct object references* lab (Access Control section, Apprentice level) and document the finding.
- What the app did: User chat transcripts were stored as static files on the server and served through predictable URLs. No ownership check.
- What I did: Changed the transcript filename, got another user's transcript, extracted a plaintext password, and logged in as carlos.

## 2. Lab details

- Lab: PortSwigger Web Security Academy - Access Control -> Insecure direct object references (Apprentice)
- URL: https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references
- Level: Apprentice (Beginner)
- My setup: Started unauthenticated. Victim account: carlos.

## 3. My findings

- OWASP Top 10 2021: A01:2021 - Broken Access Control
- CWE: CWE-639 (Authorization Bypass Through User-Controlled Key) / CWE-200 (Sensitive Info Exposure)
- Risk rating: High (8/10)
- Impact: Unauthenticated access to another user's transcript. Plaintext password exposed. Account takeover.

## 4. Tools I used

- Browser (DevTools) - for the chat page and transcript URL
- Burp Suite Community (Proxy/Intercept) - to capture the request
- Burp Repeater - to change the file number and re-send
- Browser - to log in with the stolen password and confirm

## 5. How I found it

### 5.1 Discovery

1. Opened the lab with no login.
2. Went to the Live chat tab, typed a message, clicked View transcript.
3. The URL came back as something like /download-transcript/2.txt.
4. The filename is a number the server picks, incrementing by one.
5. If 2.txt is my transcript, then 1.txt is probably someone else's.

### 5.2 Exploitation

6. Sent the request to Burp Repeater.
7. Changed 2.txt to 1.txt and sent it.
8. Got HTTP 200 with another user's chat transcript. No login, no session cookie, nothing.
9. The transcript contained a plaintext password.
10. Extracted the password from the transcript body.

### 5.3 Confirming the chain

11. Went to the main lab page and logged in as carlos with the recovered password.
12. Login worked. Lab solved. Confirmed: transcript access led straight to account takeover.

## 6. Proof of concept

### 6.1 Request

```
GET /download-transcript/2.txt HTTP/1.1
Host: <target>

```

### 6.2 Response (redacted)

```
HTTP/1.1 200 OK
Content-Type: text/plain
...

<chat transcript>
password is <VALUE_REDACTED>
...
```

- No Authorization header or session cookie needed. Fully unauthenticated.
- No 403 Forbidden or ownership check response.
- Changing 2.txt to 1.txt gave a different user's transcript.
- The transcript had a plaintext password, which is a separate credential-handling failure.

## 7. Impact

- Authentication: No login needed to get any user's transcript.
- Data exposure: Any user's chat transcript is downloadable by changing the filename.
- Credential exposure: Transcripts contain plaintext passwords.
- Account takeover: Recovered password used to log in as carlos.
- Scale: One unauthenticated request compromises one user's data and credentials.

- Plaintext passwords in chat logs are a serious credential-handling failure. Even if the app database hashes are fine, passwords that show up in chat are trivially recoverable.
- Enables credential stuffing against other services if the same password is reused.
- In a real app, the same pattern applies to any enumerable resource: exports, reports, generated documents, etc. Not just chat.

## 8. Fix

1. Server-side authorization on every resource. Verify the requester owns or is allowed to access the resource before serving it. Don't trust client-supplied filenames.
2. Unpredictable identifiers. Use random filenames or UUIDs instead of incrementing counters.
3. No plaintext passwords. Hash passwords with a strong adaptive algorithm (Argon2id, bcrypt, scrypt). Never store or log credentials in files.
4. Application-level endpoint. Serve transcripts through an app route that checks access, not a raw static file route.
5. Rate limiting and logging. Add basic monitoring on file-access endpoints to catch enumeration attempts.

## 9. What I learned

- IDOR is not just ?id= parameters. A filename like 1.txt is a direct object reference too.
- Predictable naming alone is not a vulnerability. The vulnerability is when changing the reference gives you someone else's object. Always check the ownership boundary first.
- Don't assume the session cookie means the request is authenticated. I double-checked and the transcript was open without any login.
- Read the full response body. Don't stop at the status code.
- Map the authorization boundary early. Here it was the file name, not an API parameter.
- Report the "so what?" clearly. The value is not "file 1.txt leaked" - it is "unauthenticated access to a transcript -> plaintext password -> account takeover."

## 10. References

- PortSwigger Web Security Academy - [Insecure direct object references (lab)](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)
- PortSwigger Web Security Academy - [Testing for IDORs](https://portswigger.net/burp/documentation/desktop/testing-workflow/vulnerabilities/access-controls/testing-for-idors)
- PortSwigger Web Security Academy - [Insecure direct object references (IDOR)](https://portswigger.net/web-security/access-control/idor)
- OWASP Top 10:2021 - A01:2021 Broken Access Control
- MITRE CWE-639 - Authorization Bypass Through User-Controlled Key
- MITRE CWE-200 - Exposure of Sensitive Information to an Unauthorized Actor
