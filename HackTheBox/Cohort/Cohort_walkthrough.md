## Machine Information
HTB: Cohort
Difficulty: Easy
IP: `10.129.244.174`
Machine: Linux


## Scanning and Enumeration
### Nmap Results:
```bash
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp  open  http     nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://cohort.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
443/tcp open  ssl/http nginx 1.24.0 (Ubuntu)
|_ssl-date: TLS randomness does not represent time
|_http-server-header: nginx/1.24.0 (Ubuntu)
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
|_http-title: Cohort Analytics
| ssl-cert: Subject: commonName=cohort.htb/organizationName=Cohort Analytics
| Subject Alternative Name: DNS:cohort.htb, DNS:*.cohort.htb
| Not valid before: 2026-06-01T18:47:07
|_Not valid after:  2126-05-08T18:47:07
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
### FFUF Endpoint Enumeration
```bash
ffuf -w /usr/share/wordlists/dirb/big.txt -u https://cohort.htb/FUZZ -fs 908
api                     [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 302ms]
assets                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 288ms]
status                  [Status: 403, Size: 162, Words: 4, Lines: 8, Duration: 303ms]
```
```bash
ffuf -w /usr/share/wordlists/dirb/big.txt -u https://cohort.htb/api/FUZZ -fc 405
health                  [Status: 200, Size: 42, Words: 4, Lines: 1, Duration: 1821ms]
```


## My Walkthrough
1. Using the IP I accessed the hosted http page and I accessed the endpoint `portal.html`.
2. There was a fetching algorithm that fetches any valid details it can find in the URL that is pasted in the `Source URL`.
3. Unable to find anything useful I tired `ffuf` and got some interesting endpoint one of them is `status`.
4. I tried accessing it from outside using curl but it failed when I tried accessing the endpoint with the web-age portal I got something in return
```json
{"service":"cohort-edge","status":"ok","generated_by":"nginx","upstreams":[
	{"name":"marketing","host":"cohort.htb","root":"/var/www/cohort"},
	{"name":"insights-api","host":"cohort.htb","path":"/api/","target":"127.0.0.1:5000"},
	{"name":"notebooks","host":"nb-1be3782a8afd3ad5.cohort.htb","target":"127.0.0.1:8888","note":"internal analyst workspace, not for external use"}
]}
```
5. Now as the host `nb-1be3782a8afd3ad5.cohort.htb` seems suspicious when I tried visiting that page I saw a marimo login page which asks for authentication token.
6. Now as I knew this `Cohort Analytics` can send request to its internal pages. So trying to access the internal loopback address I got an error
```json
{"ok": false, "message": "Internal or loopback addresses are not permitted."}
```
7. So I thought there might be a filter filtering the loopback address string and tried with shortening the address `127.1` or `127.0.1` with the port `8888` and it worked. Then I requested to the `/api/validate` endpoint with this command to get version information:
Request:
```json
{"url":"http://127.0.0.1:8888/terminal/ws","format":"csv"}
```
Response:
```json
{"ok": true, "fetched_status": 200, "content_type": "text/plain; charset=utf-8", "preview": "0.20.4", "message": "Source reachable."}
```
8. I got the version and just googling it gave me an exploit `CVE-2026-39987`.
> Context: This vulnerability exists in the marimo server where the Terminal WebSocket endpoint `/terminal/ws` is missing Authentication which means any unauthenticated user can send request, get a shell and execute commands on the remote server.
9. The exploit was simple to create to I created my own exploit `CVE-2026-39987-exploit.py` and tried to connect to the server using nc listener
```bash
nc -lnvp 4444
```
10. Got into the server got my `user.txt` and started looking in the system for more information.
11. Used linpeas which dumped lots of information in the terminal and one of them looked juicy
```bash
PackageKit version detected: 1.2.8-2ubuntu1.2
Vulnerable to CVE-2026-41651 (Pack2TheRoot) - PackageKit 1.2.8-2ubuntu1.2 is below the Ubuntu 24.04 fixed version: 1.2.8-2ubuntu1.5
```
12. PackageKit is a system daemon that provides a unified, distro-agonistic interface for different Linux package managers. As I looked more into the CVE mentioned I got to know that there is a Pack2Root exploit for CVE-2026-41651 avaliable.
> Context: It is a TOCTOU(Time-of-Check Time-of-Use) race condition vulnerability. A local attacker with low-level or authenticated access can time the transaction state check and use it to push unauthorized package installations. And it leads to complete root compromize.
13. Got the exploit from here <a href="https://github.com/0xBlackash/CVE-2026-41651/blob/main/CVE-2026-41651.py">CVE-2026-41651</a>.
14. Got the root and the `root.txt`. And the machine is solved ;} .