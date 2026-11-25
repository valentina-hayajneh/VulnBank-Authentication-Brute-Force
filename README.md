\# VulnBank -- Authentication Brute Force



A security testing write-up documenting reconnaissance and

authentication testing against the VulnBank web application in a lab

environment.



> \\\\\\\\\\\\\\\*\\\\\\\\\\\\\\\*Disclaimer:\\\\\\\\\\\\\\\*\\\\\\\\\\\\\\\* This report is for authorized educational/lab use

> only. Do not test systems without explicit permission.

>

> \\\\\\\\\\\\\\\*\\\\\\\\\\\\\\\*Public repository safety:\\\\\\\\\\\\\\\*\\\\\\\\\\\\\\\* Credentials and account identifiers are

> intentionally redacted. Do not publish working passwords, account

> numbers, or other sensitive data.



\## Overview



\\---



Item                                Details



\\---



Target                              `vulnbank.org`



Assessment focus                    Web login / authentication



Testing techniques                  Technology identification, service

discovery, login request

inspection, credential testing



Tools                               Wappalyzer, Nmap, Hydra,

Burp/Browser DevTools, FFUF



Outcome                             A successful login to the lab

dashboard was observed

\---



\## Methodology



\### 1\\. Technology Identification



Wappalyzer was used to identify technologies visible from the website.



The observed technologies included:



\* Cloudflare Browser Insights

\* Cloudflare CDN

\* HTTP/3



!\\\[Wappalyzer technology detection](images/01-wappalyzer.png)



\### 2\\. Network and Service Discovery



Command used:



``` bash

nmap -sV vulnbank.org

```



The scan showed the following ports exposed through Cloudflare's HTTP

proxy:



Port       Service



\\---



80/tcp     HTTP

443/tcp    HTTPS

8080/tcp   HTTP

8443/tcp   HTTPS



The scan also reported filtered TCP ports.



\*\*Note:\*\* Since the target is behind Cloudflare, these results describe

the publicly reachable proxy edge and should not be interpreted as a

direct inventory of the origin server.



!\\\[Nmap service detection](images/02-nmap.png)



\### 3\\. Login Endpoint and Response Inspection



The login endpoint was observed as:



``` text

POST https://vulnbank.org/login

```



Browser DevTools Network was used to inspect the login request and

response. An unsuccessful login returned \*\*401 Unauthorized\*\* with a

JSON response.



!\\\[Login response in DevTools](images/04-login-response-devtools.png)



\### 4\\. Credential Testing with Hydra



Hydra was used to test login credentials against the form endpoint.



Example command structure:



``` bash

hydra -L <user-list> \\\\\\\\\\\\\\\\

&#x20; -P <password-list> \\\\\\\\\\\\\\\\

&#x20; -u -t 1 -f \\\\\\\\\\\\\\\\

&#x20; vulnbank.org \\\\\\\\\\\\\\\\

&#x20; http-post-form "/login:username=^USER^\\\\\\\\\\\\\\\&password=^PASS^:F=Invalid credentials"

```



Hydra reported a candidate credential pair during testing. A reported

pair should always be manually validated because the form module's

failure condition may not correctly match the application's response.



!\\\[Hydra credential testing](images/03-hydra-attempts.png)



\### 5\\. JSON Login Testing with FFUF



The login request was also tested using FFUF with a JSON request body:



``` bash

ffuf -w <user-list>:USER \\\\\\\\\\\\\\\\

&#x20; -w <password-list>:PASS \\\\\\\\\\\\\\\\

&#x20; -X POST \\\\\\\\\\\\\\\\

&#x20; -H 'Content-Type: application/json' \\\\\\\\\\\\\\\\

&#x20; -d '{"username":"USER","password":"PASS"}' \\\\\\\\\\\\\\\\

&#x20; -u https://vulnbank.org/login \\\\\\\\\\\\\\\\

&#x20; -fc 401 \\\\\\\\\\\\\\\\

&#x20; -rate 10

```



FFUF returned a candidate with \*\*HTTP 200\*\*. The response was then

validated in the browser by successfully reaching the authenticated

dashboard.



For a public version of this repository, redact the password from the

FFUF output screenshot.



!\\\[FFUF login testing](images/05-ffuf-discovery.png)



\### 6\\. Successful Authentication



The browser displayed the VulnBank dashboard after authentication. This

confirms that the tested credentials allowed access to the application

in the lab.



Before publishing, redact the account number and any other

account-specific information visible in the screenshot. The balance may

also be redacted.



!\\\[Authenticated VulnBank](images/06-successful-login-dashboard.png)



\## Findings



\* The login endpoint accepted a credential pair that was successfully

validated by reaching the dashboard.

\* The login endpoint returned `401 Unauthorized` for an unsuccessful

login attempt observed during testing.

\* The publicly visible host was behind Cloudflare, so the Nmap results

should not be treated as a direct origin-server scan.

\* The testing demonstrated that credential guessing against the login

endpoint was possible in the observed lab setup. This does \*\*not\*\*,

by itself, establish the absence of rate limiting, account lockout,

or other protections across all conditions.



\## Recommendations



\* Use strong, unique passwords and avoid default or commonly used

credentials.

\* Apply rate limiting and progressive delays to authentication

attempts.

\* Consider account lockout or risk-based controls that avoid creating

an easy denial-of-service condition.

\* Enable multi-factor authentication where supported.

\* Monitor and alert on repeated failed login attempts.

\* Ensure authentication errors and responses do not make credential

enumeration easier.

\* Validate security controls with authorized testing and server-side

logs.



\## Evidence



Screenshots are stored in the \[`images/`](images/) directory and

numbered in the order presented in this report.



\## Disclaimer



This repository documents an educational security assessment. All

testing should be performed only on systems you own or have explicit

permission to assess.









