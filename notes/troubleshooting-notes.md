# Troubleshooting Notes – Linux Auditd Telemetry Validation Using Wazuh

## 1. Audit Rule List Initially Empty

### Observation

The command:

```bash
auditctl -l
```

returned:

```text
No rules
```

### Explanation

The audit service was active, but no runtime rules were loaded at the time of the initial check.

An active audit service does not automatically mean that a specific file or activity is being monitored.

### Resolution

A dedicated runtime watch rule was configured for the lab directory:

```bash
auditctl -w /var/tmp/lab132 -p wa -k lab132_file_activity
```

The rule was verified using:

```bash
auditctl -l
```

## 2. Old Style Watch Rule Warning

### Observation

The rule installation returned:

```text
Old style watch rules are slower
```

### Explanation

The audit utility reported a performance warning because the legacy watch-rule syntax was used.

The warning did not indicate that the rule had failed.

### Validation

The rule appeared in the loaded rule list:

```text
-w /var/tmp/lab132 -p wa -k lab132_file_activity
```

This confirmed that the rule was installed.

## 3. Understanding the Audit Rule Permissions

### Configured Rule

```bash
auditctl -w /var/tmp/lab132 -p wa -k lab132_file_activity
```

### Explanation

- `-w` specifies the path to monitor.
- `w` monitors write activity.
- `a` monitors attribute changes.
- `-k` assigns a searchable audit key.

The key was used to retrieve matching events:

```bash
ausearch -k lab132_file_activity -i
```

## 4. Understanding Multiple Audit Records

### Observation

A single file operation produced multiple audit record types.

Examples included:

- PROCTITLE
- SYSCALL
- PATH
- CWD

### Explanation

Linux Audit can generate multiple records for one event.

These records share an audit event identifier and timestamp.

### Investigation Approach

Correlate the records using the event ID and timestamp rather than treating every record as an independent operation.

Use the SYSCALL record to identify the operation and its result, and use PATH records to identify the affected file.

## 5. Understanding the Wazuh Configuration-Change Alert

### Observation

Wazuh generated:

```text
Rule ID: 80705
Rule Level: 3
Description: Auditd: Configuration changed.
```

### Explanation

The associated audit event recorded the addition of the runtime rule:

```text
op=add_rule
key=lab132_file_activity
```

The alert represented an audit configuration change.

It did not prove that the monitored file had been created, written, or modified.

### Investigation Approach

Correlate the Wazuh alert with the raw audit event and identify the process, syscall, operation, and audit event ID.

Do not interpret an audit configuration alert as a file activity alert.

## 6. Wazuh Collection Configuration

### Observation

The Wazuh configuration contained:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

### Explanation

This entry configures Wazuh to collect the Linux Audit log.

The presence of this configuration confirms the intended collection setup, but does not independently prove that every record was decoded and generated an alert.

### Investigation Approach

Compare local audit records with Wazuh events and inspect the manager log collector status.

## 7. Wazuh Integration Connection Error

### Observation

The manager log contained:

```text
Unable to run integration for custom-ticket -> integrations
Custom ticket integration failed: <urlopen error [Errno 111] Connection refused>
```

### Explanation

The configured custom-ticket integration attempted to connect to a service that refused the connection.

This was an integration issue observed during manager operation.

The available evidence did not establish that this error prevented Linux Audit log collection.

### Investigation Approach

Keep integration failures separate from audit log collection validation. Investigate the integration endpoint independently if that integration is required.

## 8. Initial Indexer Connector Warnings

### Observation

The manager log initially contained indexer connector initialization warnings.

Later messages reported successful initialization of the relevant state indices.

### Explanation

The warnings occurred during manager startup and were followed by successful initialization messages.

They should not be documented as a persistent indexer failure.

## 9. Audit Rule Removal Failure

### Observation

The following command was attempted:

```bash
auditctl -W /var/tmp/lab132 -k lab132_file_activity
```

It returned:

```text
Error sending delete rule data request (No such file or directory)
There was an error while processing parameters
```

A subsequent `auditctl -l` still showed the watch rule.

### Explanation

The attempted removal did not succeed. The rule remained visible in the loaded rule list.

The test directory was later deleted, but this did not establish that the runtime rule had been removed.

### Important Note

Do not document the audit rule cleanup as successful.

The rule-removal issue should be verified separately before considering the lab environment fully cleaned up.

## 10. Directory Removal

### Observation

The command:

```bash
rm -rf /var/tmp/lab132
```

was executed.

The subsequent check returned:

```text
ls: cannot access '/var/tmp/lab132': No such file or directory
```

### Result

The test directory was successfully removed.

However, the audit rule was still visible in the last recorded `auditctl -l` output.

Directory removal and audit rule removal must be treated as separate cleanup results.

## 11. Key Troubleshooting Lessons

- An active audit service does not guarantee that the required rules are loaded.
- A warning does not always mean that a command failed.
- Use `ausearch` to validate local audit event generation.
- Correlate related audit records using their event IDs.
- Distinguish audit configuration changes from monitored file activity.
- A Wazuh collection configuration entry does not prove that every event generated an alert.
- Investigate integration errors separately from log collection.
- Confirm rule removal instead of assuming cleanup succeeded.
- Record unresolved cleanup issues accurately.
