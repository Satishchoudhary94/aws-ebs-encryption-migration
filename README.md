# EBS Volume Encryption — Zero Downtime Guide (Steps 1–6)

> **Goal:** Encrypt an existing unencrypted EBS volume attached to a running EC2 instance using the Blue-Green approach — with zero downtime.

---

## Prerequisites

Before starting, collect your resource details:

```bash
# Get Instance ID and Availability Zone
aws ec2 describe-instances --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query 'Reservations[].Instances[].{ID:InstanceId,AZ:Placement.AvailabilityZone,State:State.Name}'

# List all attached volumes and check encryption status
aws ec2 describe-volumes \
  --filters Name=attachment.instance-id,Values=i-xxxxxxxxxxxxxxxxx \
  --query 'Volumes[].{VolumeId:VolumeId,Encrypted:Encrypted,Device:Attachments[0].Device,Size:Size}'
```

**Note down:**
- `Instance ID` → `i-xxxxxxxxxxxxxxxxx`
- `Volume ID` → `vol-xxxxxxxxxxxxxxxxx`
- `Availability Zone` → e.g., `us-east-1a`
- `Device Name` → e.g., `/dev/xvda`

---

## Step 1 — Flush & Freeze Writes (Inside EC2)

SSH into your running instance and sync all pending data to disk before snapshotting.

```bash
# SSH into the running instance
ssh -i your-key.pem ec2-user@<your-instance-ip>

# Flush all filesystem buffers to disk
sync

# (Recommended for databases) Freeze the filesystem briefly
sudo fsfreeze -f /        # Freeze — trigger snapshot immediately after this

# After snapshot is triggered (Step 2), unfreeze:
sudo fsfreeze -u /        # Unfreeze
```

> ⚠️ **Important:** Run `fsfreeze -u` immediately after triggering the snapshot. Do **not** leave the filesystem frozen for more than a few seconds.

---

## Step 2 — Create Snapshot of the Unencrypted Volume

From your local machine or AWS CloudShell:

```bash
# Create a snapshot of the unencrypted volume
aws ec2 create-snapshot \
  --volume-id vol-xxxxxxxxxxxxxxxxx \
  --description "Pre-encryption-snapshot-$(date +%Y%m%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=pre-encrypt-snap}]'
```

**Wait for the snapshot to complete:**

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids snap-xxxxxxxxxxxxxxxxx

echo "✅ Snapshot ready!"
```

> 📝 Note the `SnapshotId` from the output — you will need it in Step 3.

---

## Step 3 — Copy Snapshot With Encryption Enabled

```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-xxxxxxxxxxxxxxxxx \
  --encrypted \
  --kms-key-id alias/aws/ebs \
  --description "Encrypted-copy-$(date +%Y%m%d)" \
  --region us-east-1
```

**Wait for the encrypted snapshot to complete:**

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids snap-yyyyyyyyyyyyyyyyy

echo "✅ Encrypted snapshot ready!"
```

> 💡 **Custom KMS Key (Recommended):** Replace `alias/aws/ebs` with your own key for better access control:
> ```
> --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-key-id
> ```

> 📝 Note the new encrypted `SnapshotId` (`snap-yyy...`) — you will use it in Step 4.

---

## Step 4 — Create New AMI From the Encrypted Snapshot

```bash
aws ec2 register-image \
  --name "encrypted-ami-$(date +%Y%m%d)" \
  --architecture x86_64 \
  --root-device-name /dev/xvda \
  --virtualization-type hvm \
  --block-device-mappings '[
    {
      "DeviceName": "/dev/xvda",
      "Ebs": {
        "SnapshotId": "snap-yyyyyyyyyyyyyyyyy",
        "VolumeType": "gp3",
        "Encrypted": true,
        "DeleteOnTermination": true
      }
    }
  ]'
```

> 📝 Note the new `AMI ID` from the output (`ami-zzz...`) — you will use it in Step 5.

---

## Step 5 — Launch New EC2 Instance (With Encrypted Volume)

```bash
aws ec2 run-instances \
  --image-id ami-zzzzzzzzzzzzzzzzz \
  --instance-type <same-type-as-old-instance> \
  --key-name <your-key-pair> \
  --security-group-ids sg-xxxxxxxxx \
  --subnet-id subnet-xxxxxxxxx \
  --iam-instance-profile Name=<your-iam-profile> \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=encrypted-instance}]' \
  --count 1
```

**Wait for the new instance to be running:**

```bash
aws ec2 wait instance-running \
  --instance-ids i-newxxxxxxxxxxxxxxxxx

echo "✅ New encrypted instance is running!"
```

> 📝 Note the new `Instance ID` (`i-new...`) for use in Step 6.

---

## Step 6 — Validate the New Encrypted Instance

**Check instance status:**

```bash
aws ec2 describe-instance-status \
  --instance-ids i-newxxxxxxxxxxxxxxxxx
```

**Verify the volume is encrypted:**

```bash
aws ec2 describe-volumes \
  --filters Name=attachment.instance-id,Values=i-newxxxxxxxxxxxxxxxxx \
  --query 'Volumes[].{VolumeId:VolumeId,Encrypted:Encrypted,Device:Attachments[0].Device}'
```

Expected output:
```json
[
  {
    "VolumeId": "vol-newxxxxxxxxxxxxxxxxx",
    "Encrypted": true,
    "Device": "/dev/xvda"
  }
]
```

**SSH into the new instance and test the application:**

```bash
ssh -i your-key.pem ec2-user@<new-instance-ip>

# Run application health checks
curl http://<new-instance-ip>/health

# Check services are running
sudo systemctl status <your-service-name>
```

> ✅ Once health checks pass and the application is confirmed working, you are ready to proceed with **traffic cutover (Step 7)**.

---

## Quick Reference — Resource ID Checklist

| Resource         | Placeholder                  | Your Value |
|------------------|------------------------------|------------|
| Old Instance ID  | `i-xxxxxxxxxxxxxxxxx`        |            |
| Volume ID        | `vol-xxxxxxxxxxxxxxxxx`      |            |
| Original Snapshot| `snap-xxxxxxxxxxxxxxxxx`     |            |
| Encrypted Snapshot| `snap-yyyyyyyyyyyyyyyyy`    |            |
| New AMI ID       | `ami-zzzzzzzzzzzzzzzzz`      |            |
| New Instance ID  | `i-newxxxxxxxxxxxxxxxxx`     |            |
| Region           | `us-east-1`                  |            |
| KMS Key          | `alias/aws/ebs`              |            |

---

## Flow Summary (Steps 1–6)

```
STEP 1 → SSH in → run sync → (optionally) fsfreeze
STEP 2 → Create snapshot of unencrypted volume
STEP 3 → Copy snapshot with encryption enabled
STEP 4 → Register new AMI from encrypted snapshot
STEP 5 → Launch new EC2 instance from encrypted AMI
STEP 6 → Validate encryption + application health
         └─► Ready for traffic cutover (Step 7)
```

---

## Notes

- Keep the **old instance running** until after Step 7 (traffic swap) is complete.
- Keep the **old unencrypted snapshot** for at least 24–48 hours as a rollback option.
- If you have **multiple volumes** (e.g., root + data), repeat Steps 2–4 for each volume and attach them all when launching the new instance in Step 5.
- Snapshots taken on a running instance are **crash-consistent** — suitable for most workloads. For strict consistency, use `fsfreeze` in Step 1.
