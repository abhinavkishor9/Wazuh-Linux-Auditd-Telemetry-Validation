# Investigation Timeline 
| Time (IST) | Activity | Observation |
|---|---|---|
| 07:00:23 | Auditd service check | Service confirmed active and running |
| 07:00:23 | Audit initialization | Audit configuration loaded during service startup |
| 07:00 | Initial audit rule inspection | `auditctl -l` returned `No rules` |
| 07:00 | Audit status inspection | Auditing enabled; lost records reported as 0 |
| 07:39 | Audit log inspection | `/var/log/audit/audit.log` existed and was approximately 2.0 MB |
| 07:43:30 | Runtime audit rule added | `lab132_file_activity` rule added successfully |
| 07:43:30 | Audit configuration event | Event ID 228 recorded `op=add_rule` |
| 07:43:32 | Wazuh Discover event | Rule 80705 generated an audit configuration-change alert |
| 07:44:43 | Test file creation | Event ID 229 recorded the `touch` operation |
| 07:45:02 | Test file write | Event ID 230 recorded the shell write operation |
| 07:45:12 | Permission modification | Event ID 231 recorded the `chmod` operation |
| Later | Wazuh configuration inspection | Audit log collection entry found in `ossec.conf` |
| Later | Wazuh Manager validation | Manager active and log collector process running |
| Later | Cleanup attempt | Audit watch-rule removal command failed |
| Later | Rule verification | `auditctl -l` still displayed the lab rule |
| Later | Test directory cleanup | Directory removed successfully |

