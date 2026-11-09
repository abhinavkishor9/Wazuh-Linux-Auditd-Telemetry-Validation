# Lab 132 – Linux Auditd Telemetry Validation Using Wazuh

## Overview

This lab validates Linux Auditd telemetry generation and its collection by Wazuh.

The investigation follows controlled file activity on a Rocky Linux system and examines how Linux Audit records the activity, how the records are stored locally, and how Wazuh processes audit telemetry.

The objective is to validate the different stages of the telemetry pipeline rather than assume that an active audit service automatically guarantees successful SIEM monitoring.

## Lab Environment

| Component | Details |
|---|---|
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel Version | 5.14.0-687.31.1.el9_8.x86_64 |
| Virtualization | VMware |
| Wazuh Version | 4.13.1 |
| Audit Framework | Linux Audit |
| Audit Service | auditd |
| Audit Log | /var/log/audit/audit.log |
| Wazuh Configuration | /var/ossec/etc/ossec.conf |
| Wazuh Manager | localhost.localdomain |

## Investigation Objectives

- Validate the Linux Audit service status.
- Inspect the current audit configuration and loaded rules.
- Configure a controlled file-monitoring audit rule.
- Generate file creation, write, and permission-change activity.
- Retrieve matching audit records using `ausearch`.
- Examine SYSCALL, PATH, CWD, and PROCTITLE records.
- Validate Wazuh audit log collection configuration.
- Examine Wazuh processing and alert visibility.
- Distinguish audit configuration-change alerts from file activity events.
- Document telemetry validation results and cleanup limitations.

## Audit Monitoring Configuration

A runtime watch rule was configured for a dedicated test directory.

```bash
auditctl -w /var/tmp/lab132 -p wa -k lab132_file_activity
```

The rule monitored write and attribute changes and assigned the key `lab132_file_activity`.

The loaded rule was verified using:

```bash
auditctl -l
```

## Controlled Activity

The following operations were performed:

```bash
mkdir -p /var/tmp/lab132

touch /var/tmp/lab132/audit-test.txt

echo "Lab 132 Auditd Telemetry Validation" > /var/tmp/lab132/audit-test.txt

chmod 600 /var/tmp/lab132/audit-test.txt
```

The operations generated audit records associated with the configured key.

## Local Audit Validation

Matching records were retrieved using:

```bash
ausearch -k lab132_file_activity -i
```

The investigation identified records for:

- Audit rule configuration.
- File creation using `touch`.
- File writing through a shell operation.
- File permission modification using `chmod`.

The records contained process information, file paths, syscall details, user identity, and operation results.

## Wazuh Collection Configuration

The Wazuh configuration contained the following local file collection entry:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

This confirmed that Wazuh was configured to collect the Linux Audit log using the audit format.

The Wazuh Manager was active, and its log collector process was running.

## Wazuh Alert Analysis

The Wazuh Discover event showed:

| Field | Observed Value |
|---|---|
| Rule ID | 80705 |
| Rule Level | 3 |
| Rule Description | Auditd: Configuration changed. |
| Rule Groups | audit, audit_configuration |
| Input Type | log |
| Location | /var/log/audit/audit.log |
| Manager | localhost.localdomain |
| Event Time | Nov 9, 2026 @ 07:43:32.111 |

The underlying audit event was associated with the addition of the `lab132_file_activity` rule through `auditctl`.

The alert therefore demonstrated Wazuh processing of an audit configuration-change event. It was not evidence that separate alerts were generated for the subsequent file creation, write, and permission-change operations.

## Key Findings

- The Linux Audit service was active and enabled.
- Audit was enabled, and the initial audit rule list was empty.
- The initial audit status showed zero lost records.
- A runtime watch rule was successfully installed for `/var/tmp/lab132`.
- Linux Audit recorded the controlled file operations.
- The `ausearch` output included successful `openat` and `fchmodat` operations.
- Wazuh was configured to collect `/var/log/audit/audit.log`.
- The Wazuh Manager and log collector were running.
- Wazuh generated alert `80705` for the audit configuration change.
- Separate Wazuh alerts for the controlled file operations were not established by the supplied screenshot.

## Troubleshooting and Cleanup

The audit watch command returned the warning:

```text
Old style watch rules are slower
```

The rule was nevertheless installed and appeared in `auditctl -l`.

During cleanup, the attempted rule removal failed with:

```text
Error sending delete rule data request (No such file or directory)
There was an error while processing parameters
```

The subsequent `auditctl -l` output still showed the watch rule.

The test directory was deleted successfully, but removal of the runtime audit rule was not confirmed.

## Conclusion

This lab demonstrated successful Linux Audit event generation for controlled file activity and validated the presence of Wazuh audit log collection configuration.

Wazuh also processed an audit configuration-change event and generated alert `80705`.

However, the available Dashboard evidence did not establish separate Wazuh alerts for the file creation, write, and permission-change events.

The investigation highlights the importance of validating local event generation, log collection, decoding, rule matching, and alert visibility as separate stages.

## Skills Demonstrated

- Linux Audit Framework
- Audit Rule Configuration
- Linux File Activity Monitoring
- Audit Log Analysis
- Syscall Investigation
- Wazuh SIEM
- Log Collection Validation
- Security Event Correlation
- SOC Investigation
- Evidence-Based Documentation
