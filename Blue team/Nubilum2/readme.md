
<div align="center">

# HTB Sherlock: Nubilum 2

![Category](https://img.shields.io/badge/Category-Cloud%20Forensics-blue)
![Platform](https://img.shields.io/badge/Platform-AWS-orange)
![Focus](https://img.shields.io/badge/Focus-CloudTrail%20%7C%20IAM%20%7C%20S3-red)

**AWS CloudTrail Investigation | S3 Exposure | IAM Privilege Escalation**

</div>

## Overview

Nubilum 2 investigates a compromise of Forela's AWS environment, involving suspicious S3 activity, exposed access keys, IAM privilege escalation, and unauthorized file operations.

The investigation focuses on AWS CloudTrail logs to reconstruct the attacker's activity, identify compromised credentials, trace privilege escalation, and determine how AWS KMS encryption affected access to S3 objects.

## Tools & Artifacts

| Item | Details |
|---|---|
| Primary data source | AWS CloudTrail JSON logs |
| Aggregated log file | `Merged.json` |
| Processed logs | `filtered_events.json`, `attacker_events1.json` |
| Analysis tools | `jq`, `grep`, `sort`, `uniq`, `wc`, `less` |
| AWS services | S3, IAM, EC2, AWS KMS |
| Target bucket | `forela-fileshare` |
| AWS Region | `us-east-1` for the observed S3 operations |

## Investigation

### 1. Identify the Threat Actor

CloudTrail records were filtered for suspicious S3 API calls, including `GetObject`, `PutObject`, `ListBuckets`, and `PutBucketPolicy`.

    jq '.[].Records[] | select(.eventName == "GetObject" or .eventName == "PutObject" or .eventName == "ListBuckets" or .eventName == "PutBucketPolicy")' Merged.json > filtered_events.json

The source IP addresses were extracted and compared against the associated activity.

    cat filtered_events.json | grep -i 'sourceip' | sort | uniq

Three source IP addresses were identified:

- `35.153.199.23`
- `52.77.60.127`
- `54.242.59.197`

The investigation attributed the suspicious activity to **`54.242.59.197`**, which was associated with a Linux host and performed the relevant malicious operations.

Attacker-related events were extracted for further analysis:

    jq '.[] .Records[] | select(.sourceIPAddress == "54.242.59.197")' Merged.json > attacker_events1.json

### 2. Initial S3 Object Access

The earliest identified S3 object access in the walkthrough was an anonymous `GetObject` request against the `forela-fileshare` bucket.

| Field | Value |
|---|---|
| Event | `GetObject` |
| Timestamp | `2023-11-02T14:19:21Z` |
| Source IP | `54.242.59.197` |
| Bucket | `forela-fileshare` |
| Object | `_restricted/Priv-Keys/prod-IAM-readonly_accesskeys.csv` |
| Caller account field | `Anonymous` |
| Recipient AWS account ID | `949622803460` |

This event is significant because the object path suggests that the bucket contained sensitive IAM access-key material. The anonymous request and the broader S3 exposure are consistent with an access-control weakness.

### 3. Compromised Access Keys

Unique access-key identifiers associated with the attacker's events were counted:

    cat attacker_events1.json | grep -i 'accessKeyId' | sort -u | wc -l

**Finding:** At least **7 access keys** were identified in the filtered events.

The attacker subsequently used compromised credentials to enumerate AWS resources and perform unauthorized operations.

### 4. EC2 Discovery

The attacker issued `DescribeInstances` requests to enumerate EC2 instances.

The associated `requestParameters` revealed the filter:

| Parameter | Value |
|---|---|
| Filter name | `instance-state-name` |
| Filter value | `running` |

The event was recorded at `2023-11-02T15:05:58Z` and originated from `54.242.59.197`.

This activity demonstrates cloud resource discovery using the AWS CLI.

### 5. Failed Discovery and Privilege Escalation Attempts

CloudTrail events containing `AccessDenied` were examined to determine how many unsuccessful attempts preceded elevated access.

    grep -i 'accessdenied' attacker_events1.json | wc -l

**Finding:** The walkthrough identified **42 unsuccessful attempts**.

These failures indicate that the attacker tested permissions and AWS operations before successfully obtaining elevated privileges.

### 6. IAM Privilege Escalation

The investigation identified the IAM user **`dev-policy-specialist`** as the account associated with the successful privilege escalation.

The decisive API event was `PutUserPolicy`, which allowed an inline policy to be attached to an IAM user.

| Field | Value |
|---|---|
| Event | `PutUserPolicy` |
| IAM user | `dev-policy-specialist` |
| Policy name | `sbhyy79zky` |
| Source IP | `54.242.59.197` |

The created policy contained the following statement:

    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }

This grants all actions on all resources covered by the policy's scope. It represents an unrestricted permission grant and provides the user with administrator-equivalent access within that scope.

### 7. S3 Object Operations and KMS Encryption

The attacker's subsequent `CopyObject` activity targeted the `forela-fileshare` bucket.

The event recorded at `2023-11-02T15:26:03Z` included server-side encryption using AWS KMS.

| Field | Value |
|---|---|
| Event | `CopyObject` |
| Bucket | `forela-fileshare` |
| Source IP | `54.242.59.197` |
| Encryption | `aws:kms` |
| KMS key ARN | `arn:aws:kms:us-east-1:263954014653:key/mrk-85e24f85d964469cba9e4589335dd0f4` |
| Object referenced | `reports/Security_Briefing_Q3_2023.pptx` |

A subsequent `PutObject` event recorded the upload of another object to the same bucket.

| Field | Value |
|---|---|
| Event | `PutObject` |
| Timestamp | `2023-11-02T15:26:22Z` |
| Bucket | `forela-fileshare` |
| Uploaded object | `README2DECRYPT.txt` |
| Source IP | `54.242.59.197` |

These events establish that the attacker performed both object-copy and object-upload operations. The uploaded filename is an investigative indicator, but its name alone does not establish its contents or purpose.

### 8. Persistence Through Additional IAM Access

The attacker also created an access key for the IAM user **`forela-admin`**.

This operation is consistent with an attempt to establish additional access that could persist independently of the credentials initially compromised.

The creation of access keys for another account is a critical finding because it may allow continued access even after the original compromised credentials are revoked.

### 9. KMS Decryption Failure

The investigation also identified an authorization failure affecting the user `forela-john`.

The denied action was:

    kms:Decrypt

The user was not authorized to decrypt the protected resource associated with the ciphertext. This is relevant to the reported inability to access or download an S3 object: access to an object and permission to decrypt its contents are separate authorization requirements.

The findings therefore point to both S3 access-control concerns and KMS permissions as relevant parts of the incident.

## Attack Timeline

| Timestamp (UTC) | Activity |
|---|---|
| `2023-11-02 14:19:21` | Anonymous `GetObject` request for a sensitive-looking access-key CSV |
| `2023-11-02 15:05:58` | EC2 instance discovery through `DescribeInstances` |
| `2023-11-02 15:21:33` | `PutUserPolicy` event associated with IAM privilege escalation |
| `2023-11-02 15:26:03` | S3 `CopyObject` operation involving a KMS-encrypted object |
| `2023-11-02 15:26:22` | `PutObject` upload of `README2DECRYPT.txt` |

## Key Findings

| Finding | Evidence |
|---|---|
| Suspected threat actor | `54.242.59.197` |
| Sensitive S3 object accessed | `_restricted/Priv-Keys/prod-IAM-readonly_accesskeys.csv` |
| Minimum compromised access keys | 7 |
| Failed attempts identified | 42 |
| IAM privilege escalation | `PutUserPolicy` |
| Escalated IAM user | `dev-policy-specialist` |
| Overly permissive policy | `sbhyy79zky` |
| Additional access targeted | `forela-admin` |
| Uploaded S3 object | `README2DECRYPT.txt` |
| Unauthorized operation | `kms:Decrypt` |

## Defensive Recommendations

- **Audit S3 bucket policies:** Remove unintended public access and enforce least-privilege access to sensitive objects.
- **Rotate exposed credentials:** Revoke and replace compromised IAM access keys, including any credentials stored in S3.
- **Restrict IAM policy modification:** Limit `PutUserPolicy`, access-key creation, and other privilege-management operations to authorized administrators.
- **Review IAM permissions:** Remove wildcard `Action` and `Resource` grants unless explicitly required and tightly controlled.
- **Audit KMS key policies:** Verify that authorized principals can decrypt required objects without granting unnecessary access.
- **Investigate persistence:** Review access keys associated with `forela-admin` and other affected IAM users.
- **Strengthen CloudTrail monitoring:** Alert on unusual S3 access, repeated `AccessDenied` responses, IAM policy changes, access-key creation, and unexpected object uploads.
- **Review sensitive data exposure:** Determine whether the access-key CSV was accessible to unauthorized principals and whether any exposed credentials were used elsewhere.

## Key Takeaways

- AWS CloudTrail provides the event-level evidence needed to reconstruct cloud attacks.
- S3 exposure can reveal sensitive material that enables subsequent compromise.
- Compromised credentials can be used for cloud resource discovery and privilege escalation.
- Overly permissive IAM policies can grant broad control over cloud resources.
- S3 object permissions and AWS KMS decryption permissions must both be considered when investigating access failures.
- Monitoring IAM changes, access-key creation, and anomalous S3 operations is essential for detecting cloud compromise.

## Skills Demonstrated

`AWS CloudTrail` · `Cloud Forensics` · `S3 Security` · `IAM Analysis` · `Privilege Escalation` · `AWS KMS` · `JSON Log Analysis` · `Incident Response`
