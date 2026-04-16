**Question 1:**
Three binary packages have been placed in ``/root/binary-verification/``, with only one being authentic.
Tasks:
1. Review the official checksum in ``official-checksum.sha256``
2. Verify each binary package (v1.tar, v2.tar, v3.tar) against the official checksum
3. Determine which binary package is authentic

Complete the verification report at ``/root/binary-verification-report.txt`` with the following details:
* The official checksum value
* Verification results for each binary (AUTHENTIC or TAMPERED, along with the actual checksum)
* Identification of the authentic binary

## Solution

**Step 1 — View the official checksum**
```
cat /root/binary-verification/official-checksum.sha256
```

**Step 2 — Generate SHA256 checksum for each binary**
```
sha256sum /root/binary-verification/v1.tar
sha256sum /root/binary-verification/v2.tar
sha256sum /root/binary-verification/v3.tar
```

**Step 3: Compare and Complete the Report**
Edit the report template:
```
vi /root/binary-verification-report.txt
```
Existing File
```
Binary Verification Report
==========================
$(date)

OFFICIAL CHECKSUM:

VERIFICATION RESULTS:
v1.tar:
v2.tar:
v3.tar:

AUTHENTIC BINARY:
```

Fill the details in file
```
OFFICIAL CHECKSUM: 8739dd0797f162c7d8b87c4d3213d074f91d9cbf0bdf4cba73afa0b5becb075c
v1.tar: 9a0b036a9b0885a7521bc63c65a7baf2ce63c52ca1c86c56ff101e07762be334
v2.tar: 8739dd0797f162c7d8b87c4d3213d074f91d9cbf0bdf4cba73afa0b5becb075c
v3.tar: 9431b841b7d5201ea6687ebbba02f78ed3854613bd307fca32d871c0004f7469
AUTHENTIC BINARY: v2.tar
```






