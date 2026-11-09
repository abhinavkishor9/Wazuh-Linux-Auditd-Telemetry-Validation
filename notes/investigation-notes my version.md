# Investigation Notes
## 1. System Identification

| Attribute | Observed Value |
|---|---|
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel | 5.14.0-687.31.1.el9_8.x86_64 |
| Virtualization | VMware |
| Hostname | localhost |
| Architecture | x86-64 |

## 2. Audit Service Validation

The following command was used:

```bash
systemctl status auditd --no-pager
```

Observed results:

- Service: `auditd.service`
- State: Active (running)
- Enabled: Yes
- Main PID: 974
- Audit log: `/var/log/audit/audit.log`

The audit utility was available at:

```text
/usr/sbin/auditctl
```

The initial rule inspection returned:

```text
No rules
```

The audit status showed:

```text
enabled 1
failure 1
pid 974
rate_limit 0
backlog_limit 8192
lost 0
backlog 0
backlog_wait_time 60000
backlog_wait_time_actual 0
loginuid_immutable 0 unlocked
```

This established that auditing was enabled and no lost records were reported at the time of the check.

## 3. Runtime Audit Rule Configuration

A dedicated directory was created for controlled testing:

```bash
mkdir -p /var/tmp/lab132
```

The following runtime rule was added:

```bash
auditctl -w /var/tmp/lab132 -p wa -k lab132_file_activity
```

The command returned:

```text
Old style watch rules are slower
```

This was a performance warning, not a rule installation failure.

The rule was verified using:

```bash
auditctl -l
```

Observed rule:

```text
-w /var/tmp/lab132 -p wa -k lab132_file_activity
```

The rule monitored write and attribute changes under the test directory.

## 4. Controlled File Activity

The following operations were performed:

```bash
touch /var/tmp/lab132/audit-test.txt

echo "Lab 132 Auditd Telemetry Validation" > /var/tmp/lab132/audit-test.txt

chmod 600 /var/tmp/lab132/audit-test.txt
```

These operations provided controlled file creation, write, and permission-change activity.

## 5. Local Audit Event Analysis

Matching events were retrieved using:

```bash
ausearch -k lab132_file_activity -i
```

The output contained the following relevant audit event IDs:

| Event ID | Activity | Process | Syscall |
|---|---|---|---|
| 228 | Audit rule addition | auditctl | sendto |
| 229 | Test file creation | touch | openat |
| 230 | File write | bash | openat |
| 231 | Permission change | chmod | fchmodat |

### Event 228 – Audit Rule Addition

The event contained:

```text
op=add_rule
key=lab132_file_activity
res=yes
```

The associated process was:

```text
comm=auditctl
exe=/usr/sbin/auditctl
```

This confirmed successful addition of the runtime audit rule.

### Event 229 – File Creation

The event contained:

```text
proctitle=touch /var/tmp/lab132/audit-test.txt
```

Relevant fields included:

```text
comm=touch
exe=/usr/bin/touch
syscall=openat
success=yes
key=lab132_file_activity
```

The PATH records identified the created file:

```text
/var/tmp/lab132/audit-test.txt
```

### Event 230 – File Write

The event contained:

```text
comm=bash
exe=/usr/bin/bash
syscall=openat
success=yes
key=lab132_file_activity
```

The PATH records identified the test file involved in the operation.

### Event 231 – Permission Change

The event contained:

```text
proctitle=chmod 600 /var/tmp/lab132/audit-test.txt
```

Relevant fields included:

```text
comm=chmod
exe=/usr/bin/chmod
syscall=fchmodat
success=yes
key=lab132_file_activity
```

This established that the permission-change operation was successfully recorded.

## 6. Understanding the Audit Record Types

The test events contained multiple record types.

| Record Type | Investigation Relevance |
|---|---|
| PROCTITLE | Provides command-line information |
| SYSCALL | Identifies the system call and its result |
| PATH | Identifies the affected file or directory |
| CWD | Identifies the working directory |
| CONFIG_CHANGE | Records audit configuration activity |

Related records shared the same audit event ID and timestamp.

This is important because one operation can produce multiple records that must be correlated during investigation.

## 7. Wazuh Collection Configuration

The Wazuh configuration was inspected:

```bash
grep -n -A 8 -B 3 "/var/log/audit/audit.log" /var/ossec/etc/ossec.conf
```

The following configuration was present:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

This confirmed that Wazuh was configured to read the Linux Audit log.

The Wazuh Manager was also confirmed to be active.

Its log collector process was running:

```text
/var/ossec/bin/wazuh-logcollector
```

## 8. Wazuh Dashboard Event Analysis

The Wazuh Discover event showed:

| Field | Value |
|---|---|
| Rule ID | 80705 |
| Rule Level | 3 |
| Description | Auditd: Configuration changed. |
| Groups | audit, audit_configuration |
| Input Type | log |
| Location | /var/log/audit/audit.log |
| Manager | localhost.localdomain |
| Timestamp | Nov 9, 2026 @ 07:43:32.111 |

The raw event was associated with the audit configuration operation:

```text
op=add_rule
key=lab132_file_activity
```

The event also identified `auditctl` as the executable involved in adding the rule.

The syscall shown in the associated SYSCALL record was `sendto`. This represented communication by `auditctl` with the audit subsystem to configure the rule. It was not the file operation being monitored.

### Alert Interpretation

The Wazuh alert confirmed that an audit configuration-change event was processed and matched by rule `80705`.

It did not establish that the subsequent `touch`, shell write, or `chmod` operations generated separate Wazuh alerts.

The local audit records confirmed those file operations independently.

## 9. Wazuh Manager Log Review

The Wazuh Manager was active.

The log collector process was started during manager initialization.

The manager log also contained unrelated integration and indexer messages:

- The `custom-ticket` integration returned a connection-refused error.
- Initial indexer connector warnings appeared during startup.
- Subsequent indexer connector messages reported successful initialization.

These messages were recorded as separate operational observations. They were not sufficient to establish that audit log collection had failed.

## 10. Cleanup Observation

The following removal command was attempted:

```bash
auditctl -W /var/tmp/lab132 -k lab132_file_activity
```

It returned:

```text
Error sending delete rule data request (No such file or directory)
There was an error while processing parameters
```

A subsequent rule inspection still showed:

```text
-w /var/tmp/lab132 -p wa -k lab132_file_activity
```

The test directory was then removed:

```bash
rm -rf /var/tmp/lab132
```

The final directory check returned:

```text
ls: cannot access '/var/tmp/lab132': No such file or directory
```

The directory removal was confirmed. Removal of the runtime audit rule was not confirmed.

## 11. Investigation Findings

### Confirmed

- Auditd was active and enabled.
- Auditing was enabled.
- The initial audit rule list was empty.
- The runtime file-monitoring rule was installed.
- File creation, write, and permission-change activity generated local audit records.
- The local audit records contained process, syscall, and file-path information.
- Wazuh configuration included the audit log collection entry.
- The Wazuh Manager and log collector were running.
- Wazuh processed the audit configuration-change event and generated alert `80705`.
- The test directory was deleted.

### Not Established

- Separate Wazuh alerts for each controlled file operation.
- Removal of the runtime audit watch rule.
- Persistence of the runtime rule across reboot.
- End-to-end alert generation for every record type produced during the test.

