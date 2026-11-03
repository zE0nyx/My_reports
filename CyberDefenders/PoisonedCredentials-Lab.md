## Scenario
Your organization's security team has detected a surge in suspicious network activity. There are concerns that LLMNR (Link-Local Multicast Name Resolution) and NBT-NS (NetBIOS Name Service) poisoning attacks may be occurring within your network. These attacks are known for exploiting these protocols to intercept network traffic and potentially compromise user credentials. Your task is to investigate the network logs and examine captured network traffic.

## Questions:

### Q1: 
> In the context of the incident described in the scenario, the attacker initiated their actions by taking advantage of benign network traffic from legitimate machines. Can you identify the specific mistyped query made by the machine with the IP address 192.168.232.162?
**Approach:**Since the question just wants the mistyped query then just look at the same IP and then just filter the llmnr using the filter. And expand the `Queries`.
```text
ip.src == 192.168.232.162 and llmnr
```
<p>Answer</p>
```text
fileshaare
```


### Q2:
> We are investigating a network security incident. To conduct a thorough investigation, We need to determine the IP address of the rogue machine. What is the IP address of the machine acting as the rogue entity?
**Approach:**Now just checking all the IP requests from that the previous IP has sent or accepted from. If the machine starts receiving requests or response from different IP then that is it.
```text
llmnr && ip.addr == 192.168.232.162
``` 

<p>Answer</p>
```text
192.168.232.215
```


### Q3:
> As part of our investigation, identifying all affected machines is essential. What is the IP address of the second machine that received poisoned responses from the rogue machine?
**Approach:**Just check all the requests with the rogue IP and lookout for a new IP which the rogue machine has sent requests to.
```text
llmnr && ip.addr == 192.168.232.215
```

<p>Answer</p>
```text
192.168.232.176
```


### Q4:
> We suspect that user accounts may have been compromised. To assess this, we must determine the username associated with the compromised account. What is the username of the account that the attacker compromised?
**Approach:**As the attacker have authenticated into the system and the organization is using LLMNR and NBT-NS so it an indication that the system is using Windows and checking the NTLM authentication will give the answer. Apply the filter then expand the `Session Id` from the SMB and get the `Account` value.
```text
ntlmssp.auth.username
```

<p>Answer</p>
```text
janesmith
```


### Q5:
> As part of our investigation, we aim to understand the extent of the attacker's activities. What is the hostname of the machine that the attacker accessed via SMB?
**Approach:**To get the hostname begin with the smb traffic we get the rogue machine as a in the list checking if something is in it we get `NTLMSSP_CHALLENGE` so expanding the `DNS Computer Name` we get the computer name.

<p>Answer</p>
```text
AccountingPC
```