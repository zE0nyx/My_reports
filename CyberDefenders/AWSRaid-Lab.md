## Scenario
Your organization utilizes AWS to host critical data and applications. An incident has been reported that involves unauthorized access to data and potential exfiltration. The security team has detected unusual activities and needs to investigate the incident to determine the scope of the attack.

## Questions:

### Q1 - Compromised User Username
> Knowing which user account was compromised is essential for understanding the attacker's initial entry point into the environment. What is the username of the compromised user?
**Approach:** There were multiple login attempts that are done by the attacker before they succeeded in gaining access to a userIdentity. So to get that just need to parse out the sign-in attempts that have a `Failed authentication` then sort by the userNames.

```spl
index="aws_cloudtrail" eventSource="signin.amazonaws.com" errorMessage="Failed authentication" | stats count by userIdentity.userName | sort -count
```

<details>
<summary><b>Answer<b></summary>
```text
helpdesk.luke
```
</details>


### Q2 - Attacker Timestamp
> We must investigate the events following the initial compromise to understand the attacker's motives. What is the timestamp for the first access to an S3 object by the attacker?
**Approach:** Once the compromised username is known then we need to look at the what the attacker have downloaded or accessed. Now as CloudTrail is being used so the one to look for `GetObject` event. So using this logic in the SPL:

```spl
index="aws_cloudtrail" eventSource="s3.amazonaws.com" userIdentity.userName="helpdesk.luke" eventName="GetObject" 
| table _time, eventName, requestParameters.bucketName, requestParameteters.key, userIdentity.arn
```

<details>
<summary><b>Answer<b></summary>
```text
2023-11-02 09:55
```
</details>


### Q3 - S3 bucket name with DWG file
> Among the S3 buckets accessed by the attacker, one contains a DWG file. What is the name of this bucket?
**Approach:** So to check for the particular DWG file use the `requestParameters.key` to get the file and then fetch the `bucketName` for the particular bucket.

```spl
index="aws_cloudtrail" eventSource="s3.amazonaws.com" userIdentity.userName="helpdesk.luke" requestParameters.key="*.dwg" 
| table _time, requestParameters.bucketName, requestParameters.key
```

<details>
<summary><b>Answer<b></summary>
```text
product-designs-repository31183937
```
</details>


### Q4 - Misconfigured S3 bucket
> We've identified changes to a bucket's configuration that allowed public access, a significant security concern. What is the name of this particular S3 bucket?
**Approach:** Since we are looking for bucket changes that allowed public access there are only two eventName `GetBucketPublicAccessBlock` or `GetAccountPublicAccessBlock` then just checking the bucketName that are modified after the initial access happened gave the name.

```spl
index="aws_cloudtrail" eventSource="s3.amazonaws.com" userIdentity.userName="helpdesk.luke" eventName="GetBucketPublicAccessBlock" 
| table _time, requestParameters.bucketName 
| sort _time
```

<details>
<summary><b>Answer<b></summary>
```text
backup-and-restore98825501
```
</details>


### Q5 - Username creation by attacker
> Creating a new user account is a common tactic attackers use to establish persistence in a compromised environment. What is the username of the account created by the attacker?
**Approach:** To check for the user account creation check the eventName `CreateUser` request under the `iam` subdomain that will give any additions to the users.

```spl
index="aws_cloudtrail" eventSource="iam.amazonaws.com" userIdentity.userName="helpdesk.luke" eventName="CreateUser" 
| table requestParameters.userName
```

<details>
<summary><b>Answer<b></summary>
```text
marketing.mark
```
</details>


### Q6 - User group
> Following account creation, the attacker added the account to a specific group. What is the name of the group to which the account was added?
**Approach:** Whenever a user is added to a group a event is called known as `AddUserToGroup` and fetching the `groupName` from the `requestParameters` gave the result.

```spl
index="aws_cloudtrail" eventSource="iam.amazonaws.com" userIdentity.userName="helpdesk.luke" eventName="AddUserToGroup"
| table requestParameters.groupName, requestParameters.userName
```

<details>
<summary><b>Answer<b></summary>
```text
Admins
```
</details>