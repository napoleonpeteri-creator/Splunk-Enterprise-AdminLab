=======================
SPLUNK ADMINLAB
TROUBLESHOOTING DOCUMENTATION
HF → INDEXER CUSTOM INDEX INGESTION FAILURE
============================================================

PROJECT:
Splunk Enterprise Administration Lab

COMPONENTS:
- Splunk Heavy Forwarder (HF): Ubuntu VM
- Splunk Enterprise / Indexer: Windows Host
- Data Source: HDFS_raw_90MB.log
- Application: AdminLab
- Sourcetype: adminlab:hdfs
- Destination Index: adminlab
- Forwarding Port: TCP/9997

============================================================
1. PROBLEM STATEMENT
============================================================

The AdminLab Heavy Forwarder was successfully configured to
monitor the HDFS dataset:

/opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

The HF was configured to forward data to the Splunk Enterprise
Indexer at:

10.210.116.152:9997

The forwarding architecture had previously been proven to work
when the input used:

index = main

However, when the input was changed to:

index = adminlab

the Indexer did not return the expected events when searching:

index=adminlab

This created the initial suspicion that the HF → Indexer
forwarding path might be broken.

============================================================
2. INITIAL EVALUATION
============================================================

Rather than immediately performing extensive network and log
diagnostics, the existing evidence was evaluated first.

A critical observation was made:

HF → Indexer → main = WORKING

HF → Indexer → adminlab = NOT WORKING

This meant the basic forwarding architecture was already proven.

Therefore, repeatedly testing:

- TCP/9997
- network connectivity
- forwarding connectivity
- Splunk daemon status
- metrics logs
- splunkd logs

would not be the most logical first approach.

The investigation was narrowed to the difference between
the working configuration and the failing configuration.

The primary difference was the destination index.

============================================================
3. INDEXER-SIDE VERIFICATION
============================================================

The Splunk Web interface on the Windows Splunk Enterprise
instance was opened:

http://10.210.116.152:8000

The Indexes page was inspected.

The Indexer contained multiple indexes, including:

main
audit
_internal
metrics
etc.

However, the custom:

adminlab

index was not present initially.

This was the key finding.

The HF was configured to send events to:

index = adminlab

but the Indexer did not yet have the corresponding custom
destination index.

============================================================
4. SOLUTION
============================================================

The correct approach was to create the destination index on
the Indexer rather than continuing to diagnose the forwarding
network.

Using Splunk Web:

Settings
    ↓
Indexes
    ↓
New Index

The following index was created:

Index Name:
adminlab

The index was successfully created and showed:

Status: Active
Type: Events
Event Count: 0

At this point the Indexer had a valid destination for events
sent by the Heavy Forwarder.

============================================================
5. HF CONFIGURATION CHANGE
============================================================

The Heavy Forwarder configuration was then changed so that
the AdminLab HDFS monitor would send events to the newly
created index.

Configuration file:

/opt/splunk/etc/apps/AdminLab/local/inputs.conf

Before:

index = main

After:

index = adminlab

============================================================
6. BACKUP BEFORE MODIFICATION
============================================================

Before modifying the configuration, a backup was created:

cd /opt/splunk/bin

cp /opt/splunk/etc/apps/AdminLab/local/inputs.conf \
   /opt/splunk/etc/apps/AdminLab/local/inputs.conf.backup

PURPOSE:

This created a rollback copy of inputs.conf before making the
configuration change.

If the modification caused an unexpected problem, the
original configuration could be restored.

============================================================
7. CHANGE THE INDEX
============================================================

The destination index was changed with:

sed -i 's/^index = main$/index = adminlab/' \
   /opt/splunk/etc/apps/AdminLab/local/inputs.conf

PURPOSE:

This command searches for the exact configuration line:

index = main

and changes it to:

index = adminlab

The command was deliberately limited to the exact line so that
other configuration values would not be modified.

============================================================
8. VERIFY EFFECTIVE SPLUNK CONFIGURATION
============================================================

Instead of assuming that the file modification was sufficient,
Splunk's effective configuration was verified using btool:

./splunk btool inputs list --debug | grep -A6 -B2 'HDFS_raw_90MB.log'

The resulting configuration showed:

[monitor:///opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log]

index = adminlab

sourcetype = adminlab:hdfs

This confirmed that Splunk was actually interpreting the
configuration as intended.

============================================================
9. RESTART THE HEAVY FORWARDER
============================================================

The Heavy Forwarder was restarted:

./splunk restart

The restart completed successfully.

The Splunk daemon started successfully and the HF returned to
an operational state.

============================================================
10. CONTROLLED TEST EVENT
============================================================

Because Splunk had already processed the existing 90 MB file,
simply restarting Splunk would not necessarily cause the
existing data to be indexed again.

Therefore, a controlled new event was appended to the monitored
file.

The correct command was:

echo "ADMINLAB_TEST_EVENT $(date '+%Y-%m-%d %H:%M:%S')" >> \
/opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

PURPOSE:

This created new data at the end of the already monitored file.

The new line allowed the ingestion pipeline to be tested
without re-ingesting the entire 90 MB dataset.

============================================================
11. INDEXER-SIDE VALIDATION
============================================================

The Splunk Web interface on the Windows Indexer was used to
search:

index=adminlab "ADMINLAB_TEST_EVENT"

The search successfully returned events.

The returned events showed:

host = Splunk-HF

source =
/opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log

sourcetype = adminlab:hdfs

index = adminlab

This provided direct evidence that the events had travelled
through the complete ingestion pipeline.

============================================================
12. FINAL DATA FLOW
============================================================

The final working architecture is:

HDFS_raw_90MB.log
        |
        v
Splunk Heavy Forwarder
        |
        | monitor input
        |
        | index = adminlab
        |
        v
TCP/9997
        |
        v
Splunk Enterprise Indexer
        |
        v
adminlab index
        |
        v
Splunk Search

============================================================
13. ROOT CAUSE
============================================================

The root cause was a destination-index configuration problem.

The Heavy Forwarder was configured to send the AdminLab data
to:

index = adminlab

but the corresponding custom adminlab index had not been
properly created/available on the Indexer.

The underlying HF → Indexer forwarding path was not the
problem.

This was demonstrated by the fact that the same forwarding
architecture had previously successfully delivered data to:

index = main

Once the Indexer-side adminlab index was created and the HF
input was configured to use that index, new events were
successfully indexed and searchable.

============================================================
14. IMPORTANT LESSONS LEARNED
============================================================

LESSON 1: FOLLOW THE EVIDENCE

When one destination works and another does not, compare the
differences between the working and failing configurations
before performing broad diagnostics.

In this case:

main = worked
adminlab = failed

The destination index became the logical focus.

------------------------------------------------------------

LESSON 2: A WORKING FORWARDING PATH IS IMPORTANT EVIDENCE

Because:

HF → Indexer → main

was already working, the existence of a forwarding problem
should not have been assumed.

This prevented unnecessary network troubleshooting.

------------------------------------------------------------

LESSON 3: THE DESTINATION INDEX MUST EXIST ON THE INDEXER

Configuring:

index = adminlab

on the HF does not by itself create the destination index on
the receiving Indexer.

The Indexer must have the appropriate index configured.

------------------------------------------------------------

LESSON 4: USE btool TO VERIFY EFFECTIVE CONFIGURATION

The configuration file on disk is not the only thing that
matters.

The important question is:

"What configuration is Splunk actually using?"

The command:

./splunk btool inputs list --debug

was used to verify the effective configuration.

This confirmed:

index = adminlab

and:

sourcetype = adminlab:hdfs

for the intended monitor stanza.

------------------------------------------------------------

LESSON 5: BACK UP CONFIGURATION BEFORE MODIFYING IT

Before changing inputs.conf:

cp inputs.conf inputs.conf.backup

was used.

This establishes a simple rollback mechanism and is a good
administrative practice.

------------------------------------------------------------

LESSON 6: RESTARTING DOES NOT MEAN OLD DATA WILL BE REINDEXED

The existing 90 MB file had already been processed.

Restarting Splunk does not automatically mean the entire file
will be indexed again.

A controlled new event was therefore appended to the file.

------------------------------------------------------------

LESSON 7: CONTROLLED TEST DATA IS BETTER THAN GUESSWORK

Instead of repeatedly searching logs or running numerous
diagnostic commands, a unique test event was generated:

ADMINLAB_TEST_EVENT

The event was then searched directly in:

index=adminlab

This provided a clean end-to-end validation.

------------------------------------------------------------

LESSON 8: CHANGE ONE VARIABLE AT A TIME

The investigation eventually followed this controlled sequence:

1. Confirm existing behavior.
2. Identify the difference.
3. Create the missing destination index.
4. Change the HF destination.
5. Verify the effective configuration.
6. Restart the HF.
7. Generate one controlled event.
8. Search for that event on the Indexer.

This makes troubleshooting easier to reason about and reduces
the possibility of introducing additional problems.

============================================================
15. FINAL VALIDATION RESULT
============================================================

RESULT:

SUCCESS

The controlled test event was successfully received and
searchable in:

index=adminlab

Evidence:

host = Splunk-HF
source = /opt/splunk/etc/apps/AdminLab/datasets/HDFS/HDFS_raw_90MB.log
sourcetype = adminlab:hdfs

Therefore:

HF monitoring              = PASS
HF index assignment        = PASS
HF forwarding              = PASS
Indexer destination index  = PASS
Indexer ingestion          = PASS
Searchability              = PASS

============================================================
16. ADMINISTRATIVE TAKEAWAY
============================================================

This troubleshooting exercise demonstrated an important
Splunk administration principle:

When data reaches a Splunk Indexer under one index but fails
under another, do not immediately assume that the forwarding
layer is broken.

First compare:

SOURCE
INPUT
INDEX
SOURCETYPE
DESTINATION

A successful data path to "main" provided a baseline.
The difference introduced by "adminlab" narrowed the problem
to the custom index configuration.

The issue was resolved by properly creating the destination
index on the Indexer and aligning the Heavy Forwarder's monitor
configuration with that destination.

============================================================
END OF INCIDENT DOCUMENTATION
=====================================
