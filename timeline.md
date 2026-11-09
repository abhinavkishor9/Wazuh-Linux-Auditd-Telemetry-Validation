# Investigation Timeline – Lab 19: File Signature and Magic Byte Analysis

## Environment Preparation

- Created the Lab 19 workspace under `/home/abhinav/malware-yara-labs/lab19`.
- Created the `samples`, `tools`, `evidence`, and `output` directories.
- Verified the availability of `file`, `xxd`, `hexdump`, and `sha256sum`.
- Confirmed Python 3.12.3 was available.

## Sample Generation

- Created `tools/create_samples.py`.
- Generated seven controlled synthetic files.
- Verified that all seven files existed in the `samples` directory.
- Confirmed their individual file sizes.

## File Identification

- Executed `file samples/*`.
- Identified PDF and JPEG samples.
- Observed generic `data` classifications for the minimal ZIP- and PNG-signature samples.
- Observed ASCII text classification for the MZ-prefix sample.
- Identified `report.exe` as a PDF document.

## Magic Byte Inspection

- Inspected the first 32 bytes of each sample using `xxd`.
- Corrected an initial Bash loop syntax error.
- Corrected the use of `tee` so the command output was properly saved.
- Preserved the hexadecimal results in `evidence/magic-byte-analysis.txt`.

## Cross-Validation

- Inspected the same sample headers using `hexdump`.
- Compared the displayed hexadecimal bytes with the earlier `xxd` output.
- Confirmed that both tools showed consistent header data.

## Extension Mismatch Analysis

- Identified a PDF signature in `report.exe`.
- Identified a ZIP signature in `invoice.pdf`.
- Documented the MZ prefix in `program.exe` and the limitations of the synthetic sample.
- Distinguished signature observations from complete file-format validation.

## Evidence Collection

- Generated SHA-256 hashes for all seven samples.
- Saved the results in `evidence/sample-hashes.txt`.
- Saved file-identification results in `evidence/file-identification.txt`.
- Verified the contents of the evidence directory.

## Final Status

The planned signature-inspection workflow was completed. All seven samples were examined, their leading byte patterns were recorded, and the extension mismatches were documented without making unsupported malware classifications.
