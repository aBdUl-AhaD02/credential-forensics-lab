# Active Directory Credential Forensics Lab

## Description

A TryHackMe lab investigating an insider threat scenario: two employees were fired, but
their accounts were never deactivated. The company suspected the departed employees used
their still-active credentials to steal private files from the server, and the task was to
build concrete forensic proof from the available evidence — a network packet capture and a
memory dump of the LSASS process.

The investigation covers the full chain: identifying the exfiltrated file from network
traffic, extracting and validating NTLM authentication hashes, cracking a weak password,
and confirming active credential sessions through memory forensics. The end result ties
stolen credentials directly to the file exfiltration, proving the theory with evidence
rather than assumption.

**Category:** Digital Forensics / Incident Response
**Platform:** TryHackMe
**Tools:** Wireshark, Hashcat, Python, pypykatz

---

## 1. Identifying the Exfiltrated File

Loaded the network capture into Wireshark and filtered for SMB2 traffic to trace the file
transfer activity. Used File > Export Objects to pull the exact file that was taken off
the server.


## 2. Extracting NTLM Authentication Hashes

Filtered on `ntlmssp` to isolate the authentication exchange for both compromised accounts,
capturing the Domain, Host, Session Key, NTProofStr, and NT response values needed to
reconstruct each session.


## 3. Cracking the Password

Ran the extracted NT hash through Hashcat against the rockyou wordlist:

```bash
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt
```

The password cracked in under 30 seconds, confirming the account's credentials were weak
enough to be recovered offline.


## 4. Manually Validating the NTLMv2 Handshake

Rather than relying solely on automated tooling, I wrote a Python script to manually
recompute the HMAC-MD5 values (NTProofStr and Key Exchange Key) from the cracked password
and compare them against the values captured on the wire.

```bash
python3 hash.py --user mrealman --domain WORKGROUP --password <cracked_password> \
  --ntproofstr <captured_ntproofstr> --key <captured_session_key> -v
```

The computed NTProofStr matched the captured value exactly, confirming the password was
authentic to that session.


Repeated the process for the second compromised account:

```bash
python3 hash2.py --user eshellstrop --domain WORKGROUP --ntlmhash <ntlm_hash> \
  --ntproofstr <captured_ntproofstr> --key <captured_session_key>
```

## 5. Memory Forensics with pypykatz

Analyzed the LSASS memory dump using pypykatz, a pure-Python Mimikatz alternative, to
extract logon sessions, NT hashes, and Kerberos keys without needing to run a native
Mimikatz binary on the host.

```bash
pip install pypykatz
pypykatz lsa minidump lsass.DMP | grep -i -C15 mrealman
pypykatz lsa minidump lsass.DMP | grep -i -C15 eshellstrop
```

Both accounts showed active logon sessions on the compromised host at the time the file
was transferred.

---

## Findings

- Both terminated employee accounts remained active and were used to authenticate to the
  server after termination.
- Network traffic confirmed a private file was transferred using one of these sessions.
- NTLM hash cracking and manual NTLMv2 validation confirmed the credentials were valid and
  matched the captured session data.
- Memory forensics via pypykatz corroborated active logon sessions for both accounts at
  the time of the incident.

## Key Takeaway

This incident traces back to a single missed step: offboarding. Accounts should be
deactivated the moment an employee is terminated, not on a delayed schedule. Combining
network traffic analysis with memory forensics provided a complete, evidence-backed chain
from credential misuse to data exfiltration.
