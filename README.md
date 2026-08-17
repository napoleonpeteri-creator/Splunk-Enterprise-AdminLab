# Splunk Enterprise AdminLab

## Hands-On Splunk Enterprise Administration Laboratory

This repository documents my hands-on work building and administering
a Splunk Enterprise environment from the ground up.

The lab focuses on practical Splunk administration rather than
theoretical configuration alone.

### Current Architecture

```text
Windows Splunk Enterprise Indexer
              ▲
              │ TCP 9997
              │
Ubuntu Splunk Heavy Forwarder
              ▲
              │
       Custom HDFS Dataset




PROJECT 1 – CUSTOM DATA SOURCE / MONITORED INPUT
============================================================

OBJECTIVE:
Rebuild the Splunk Heavy Forwarder on a fresh Ubuntu VM, recreate
the AdminLab application, transfer a controlled raw HDFS dataset,
configure Splunk to monitor the file, forward the data to the
Windows Splunk Indexer, and verify the resulting events.

The purpose of this module is to establish a BASELINE before any
advanced parsing configuration is introduced.

The HDFS dataset is intentionally ingested in its raw form so that
the effect of later parsing configurations can be observed through
a controlled BEFORE / AFTER comparison.

============================================================
1. LAB ARCHITECTURE
============================================================

WINDOWS SOURCE
Original HDFS.log
~1.54 GB
        |
        | Create controlled 90 MiB raw copy
        v
HDFS_raw_90MB.log
        |
        | SCP / SSH
        v
UBUNTU SPLUNK HEAVY FORWARDER
<HF-IP>
        |
        | inputs.conf
        |
        | TCP 9997
        v
WINDOWS SPLUNK ENTERPRISE INDEXER
<INDEXER-IP>
        |
        v
index=main
sourcetype=adminlab:hdfs
        |
        v
672,927 events

NOTE:
Actual internal IP addresses are intentionally replaced with
<HF-IP>, <INDEXER-IP>, and <HOST-IP> in this public GitHub
documentation.

============================================================
2. LAB ENVIRONMENT
============================================================

HEAVY FORWARDER:

Operating System:
Ubuntu Linux

Role:
Splunk Heavy Forwarder

Splunk Home:
/opt/splunk

INDEXER:

Operating System:
Windows

Role:
Splunk Enterprise Indexer

Receiving Port:
TCP 9997

DATA SOURCE:

Original file:
HDFS.log

Original size:
1,577,982,906 bytes

Approximate size:
1.54 GB

CONTROLLED LAB COPY:

File:
HDFS_raw_90MB.log

Size:
94,371,840 bytes

Approximate size:
90 MiB

============================================================
3. DATA-SAFETY / LICENSING CONTROL
============================================================

The original HDFS dataset was approximately 1.54 GB.

The complete original dataset was NOT transferred into Splunk.

A controlled 90 MiB raw copy was created instead.

This provides a significant safety margin below the 100 MB
ingestion target established for this lab.

The original HDFS.log remained untouched.

The sample was not:

- Cleaned
- Parsed
- Rewritten
- Timestamp-modified
- Field-extracted
- Preprocessed

The purpose was to observe how Splunk initially handles the raw
HDFS data before parsing configurations are introduced.

============================================================
4. REBUILD THE ADMINLAB APPLICATION
============================================================

The Heavy Forwarder was a NEW Ubuntu VM with a FRESH Splunk
installation.

No previous AdminLab files were assumed to exist.

Directories were created:

mkdir -p /opt/splunk/etc/apps/AdminLab/default
mkdir -p /opt/splunk/etc/apps/AdminLab/local
mkdir -p /opt/splunk/etc/apps/AdminLab/datasets/HDFS
mkdir -p /opt/splunk/etc/apps/AdminLab/working/HDFS

Ownership was assigned to Splunk:

chown -R splunk:splunk /opt/splunk/etc/apps/AdminLab

Final structure:

AdminLab/
├── default/
│   └── app.conf
├── local/
├── datasets/
│   └── HDFS/
└── working/
    └── HDFS/

============================================================
5. CREATE app.conf
============================================================

FILE:

/opt/splunk/etc/apps/AdminLab/default/app.conf

CONTENT:

[install]
is_configured = 1

[ui]
is_visible = 1
label = AdminLab

[launcher]
author = AdminLab
description = Splunk Enterprise Administration Lab
version = 1.0.0

============================================================
6. CREATE CONTROLLED HDFS SAMPLE
============================================================

The original Windows HDFS.log was preserved.

A 90 MiB raw byte-level copy was created.

The original source:

C:\Users\Devia\Documents\SPLUNK-TUTORIAL_MATERIALS\PRACTISE_GITHUB_DOMAIN\HDFS_v1\HDFS.log

The controlled copy:

HDFS_raw_90MB.log

Size:

94,371,840 bytes

The controlled copy was created without modifying the contents.

The purpose was to provide a manageable dataset for the Splunk
ingestion experiment while preserving the original source.

============================================================
7. INSTALL SSH ON THE NEW HEAVY FORWARDER
============================================================

The fresh Ubuntu installation did not initially have the SSH
server installed.

Command:

apt update && apt install -y openssh-server

Enable and start SSH:

systemctl enable --now ssh

Verify:

systemctl status ssh --no-pager

RESULT:

SSH service:
active (running)

SSH was listening on TCP port 22.

============================================================
8. VERIFY WINDOWS → HF SSH CONNECTIVITY
============================================================

From Windows PowerShell:

Test-NetConnection <HF-IP> -Port 22

RESULT:

TcpTestSucceeded : True

This confirmed that the Windows machine could reach the Ubuntu
Heavy Forwarder over TCP port 22.

============================================================
9. SSH AUTHENTICATION TROUBLESHOOTING
============================================================

INITIAL SCP RESULT:

Permission denied, please try again.

The network connection was already confirmed to be working, so
the investigation focused specifically on SSH authentication.

Effective SSH configuration was checked:

sshd -T | grep -E 'permitrootlogin|passwordauthentication|usepam'

Relevant result:

permitrootlogin yes

SSH configuration syntax was validated:

sshd -t

No configuration error was returned.

Root account status was checked:

passwd -S root

The root account was found to be locked.

The root password was reset:

passwd root

SSH was restarted:

systemctl restart ssh

SSH service was verified:

systemctl is-active ssh

RESULT:

active

After correcting root authentication, SCP was retried.

============================================================
10. SCP DATA TRANSFER
============================================================

SYSTEM:
Windows PowerShell

SOURCE:

C:\Users\Devia\Documents\SPLUNK-TUTORIAL_MATERIALS\PRACTISE_GITHUB_DOMAIN\HDFS_v1\HDFS_raw_90MB.log

DESTINATION:

root@<HF-IP>:/opt/splunk/etc/apps/AdminLab/datasets/HDFS/

EXACT SCP COMMAND USED:

scp "C:\Users\Devia\Documents\SPLUNK-TUTORIAL_MATERIALS\PRACTISE_GITHUB_DOMAIN\HDFS_v1\HDFS_raw_90MB.log" root@<HF-IP>:/opt/splunk/etc/apps/AdminLab/datasets/HDFS/

COMMAND BREAKDOWN:

scp
Secure Copy Protocol used to transfer the file over SSH.

"C:\Users\Devia\...\HDFS_raw_90MB.log"
The source file on the Windows machine.

root@<HF-IP>
The root account and IP address of the Ubuntu Heavy Forwarder.

:/opt/splunk/etc/apps/AdminLab/datasets/HDFS/
The destination directory on the Heavy Forwarder.

INITIAL RESULT:

Permission denied, please try again.

TROUBLESHOOTING:

- Confirmed TCP/22 connectivity.
- Confirmed ssh.service was active.
- Checked effective SSH configuration.
- Validated sshd configuration.
- Checked root account status.
- Root account was found locked.
- Reset root password.
- Restarted SSH.
- Retried SCP.

FINAL RESULT:

Transfer succeeded.

TRANSFER RESULT:

HDFS_raw_90MB.log
100%
~90 MB
~27.0 MB/s
~00:03

============================================================
11. VERIFY DATASET ON HEAVY FORWARDER
============================================================

The transferred file was verified:

ls -lh /opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

The file was approximately:

90M

Ownership was corrected:

chown splunk:splunk /opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

Final ownership:

splunk splunk

This ensured that the Splunk service account could access the
dataset.

============================================================
12. CONFIGURE inputs.conf
============================================================

FILE:

/opt/splunk/etc/apps/AdminLab/local/inputs.conf

CONFIGURATION:

[monitor:///opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log]
disabled = false
index = main
sourcetype = adminlab:hdfs

Ownership:

chown splunk:splunk /opt/splunk/etc/apps/AdminLab/local/inputs.conf

============================================================
13. VALIDATE inputs.conf USING btool
============================================================

Command:

/opt/splunk/bin/splunk btool inputs list --debug | grep -A 5 -B 1 'HDFS_raw_90MB.log'

RESULT:

Splunk successfully recognized the AdminLab input configuration.

The configuration was loaded from:

/opt/splunk/etc/apps/AdminLab/local/inputs.conf

The expected monitor stanza was confirmed:

[monitor:///opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log]
disabled = false
index = main
sourcetype = adminlab:hdfs

============================================================
14. RESTART HEAVY FORWARDER
============================================================

Splunk was restarted:

su - splunk -s /bin/bash -c '/opt/splunk/bin/splunk restart'

Status was verified:

/opt/splunk/bin/splunk status

RESULT:

splunk is running

Splunk helpers are running

============================================================
15. VERIFY INPUT PROCESSING
============================================================

Command:

/opt/splunk/bin/splunk list inputstatus

The HDFS input reported:

file position = 94371840
file size     = 94371840
percent       = 100.00
type          = done reading (batch)

INTERPRETATION:

The Heavy Forwarder successfully read the entire 90 MiB dataset.

This confirmed that the monitor input was active and functioning.

============================================================
16. VERIFY DATA ON INDEXER
============================================================

SYSTEM:

Windows Splunk Enterprise Indexer

SEARCH:

index=main sourcetype="adminlab:hdfs"
| stats count min(_time) as earliest max(_time) as latest

RESULT:

672,927 events

This confirmed successful end-to-end ingestion.

============================================================
17. VERIFY HOST / SOURCE / SOURCETYPE / RAW EVENTS
============================================================

The indexed events were inspected using Splunk Search & Reporting.

HOST:

Splunk-HF

SOURCE:

/opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

SOURCETYPE:

adminlab:hdfs

INDEX:

main

RAW EVENT:

Raw HDFS event content was visible in the Events view.

Example raw event pattern observed:

INFO dfs.DataNode$PacketResponder...

The Events view confirmed that the original HDFS log content was
successfully received and indexed.

============================================================
18. COMPLETE DATA FLOW
============================================================

Windows HDFS source
        |
        | Create controlled raw 90 MiB sample
        v
HDFS_raw_90MB.log
        |
        | SCP
        v
Ubuntu Splunk Heavy Forwarder
        |
        | inputs.conf
        |
        | monitor://
        v
Splunk ingestion
        |
        | TCP 9997
        v
Windows Splunk Enterprise Indexer
        |
        v
index=main
        |
        v
sourcetype=adminlab:hdfs
        |
        v
672,927 events
        |
        v
Search / Raw Event Verification

============================================================
19. TROUBLESHOOTING SUMMARY
============================================================

PROBLEM 1:
SCP returned:

Permission denied, please try again.

CAUSE:

The root account was locked and SSH authentication was therefore
failing even though TCP/22 connectivity and the SSH service were
working.

INVESTIGATION:

Test-NetConnection <HF-IP> -Port 22

systemctl status ssh --no-pager

sshd -T | grep -E 'permitrootlogin|passwordauthentication|usepam'

sshd -t

passwd -S root

RESOLUTION:

passwd root

systemctl restart ssh

The SCP command was retried successfully.

------------------------------------------------------------

PROBLEM 2:
The transferred HDFS file was owned by root.

RESOLUTION:

chown splunk:splunk /opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

RESULT:

The Splunk service account became the owner.

------------------------------------------------------------

PROBLEM 3:
Need to verify that Splunk actually consumed the monitored file.

RESOLUTION:

/opt/splunk/bin/splunk list inputstatus

RESULT:

file position = 94371840
file size     = 94371840
percent       = 100.00
type          = done reading (batch)

============================================================
20. SPLUNK ADMINISTRATION SKILLS DEMONSTRATED
============================================================

This module provided hands-on experience with:

- Splunk Enterprise
- Splunk Heavy Forwarder
- Splunk Indexer
- Splunk app structure
- app.conf
- inputs.conf
- Monitor inputs
- Sourcetype assignment
- Index selection
- btool
- Input status
- Splunk service management
- Linux permissions
- Linux ownership
- SSH administration
- SCP
- Data onboarding
- Data forwarding
- Indexer verification
- Raw event inspection
- Troubleshooting
- End-to-end ingestion validation

============================================================
21. ADMINISTRATIVE LESSONS
============================================================

Successful Splunk data onboarding requires more than creating an
input.

An administrator must be able to determine:

WHERE:
Did the data come from?

HOW:
Did the data reach Splunk?

WHAT:
Sourcetype was assigned?

WHERE:
Was the data indexed?

WHO:
Owns and accesses the source file?

DID:
Did Splunk actually consume the file?

CAN:
The resulting events be searched and verified?

The complete workflow demonstrated was:

SOURCE
   ↓
TRANSFER
   ↓
PERMISSIONS
   ↓
INPUT
   ↓
FORWARDING
   ↓
INDEXING
   ↓
SEARCH
   ↓
VERIFICATION

============================================================
22. MODULE 1 RESULTS
============================================================

STATUS:

MODULE 1 COMPLETE ✓

REQUIREMENTS:

New Heavy Forwarder:
✓

AdminLab application:
✓

AdminLab directory structure:
✓

Original HDFS dataset preserved:
✓

Controlled 90 MiB raw sample:
✓

SSH installed:
✓

SSH connectivity:
✓

SCP transfer:
✓

File ownership:
✓

inputs.conf:
✓

btool validation:
✓

Monitor input:
✓

Input processing:
✓

HF → Indexer forwarding:
✓

Indexer ingestion:
✓

Host verification:
✓

Source verification:
✓

Sourcetype verification:
✓

Raw event verification:
✓

Indexed events:
672,927
