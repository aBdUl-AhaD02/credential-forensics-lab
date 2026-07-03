#!/bin/bash
# Sets up the AD credential forensics lab repo and pushes it to GitHub.
# Run this from wherever you want the project folder created.

set -e

REPO_NAME="ad-credential-forensics-lab"
GITHUB_USER="aBdUl-AhaD02"
SCREENSHOTS_DIR="$HOME/Desktop/evidence-1697996360986"  # change this to wherever your screenshots actually live

echo "Setting up $REPO_NAME..."

mkdir -p "$REPO_NAME/screenshots"
cd "$REPO_NAME"

# grab the screenshots - adjust the source path above if yours are somewhere else
cp "$SCREENSHOTS_DIR"/*.png screenshots/ 2>/dev/null || echo "No screenshots found at $SCREENSHOTS_DIR — copy them in manually before pushing."

# write the README
cat > README.md << 'EOF'
# Active Directory Credential Forensics Lab

A TryHackMe lab writeup on tracking down two former employees who used their still-active
credentials to pull private files off a company server after being let go.

The scenario: a junior sysadmin forgot to deactivate two accounts. Evidence available is a
small network capture and an LSASS memory dump. That's enough to prove exactly what happened.

## Tools used

Wireshark, Hashcat, Python (for manual NTLMv2 hash validation), pypykatz

## Part 1 — Finding the stolen file in the packet capture

Opened the capture in Wireshark, filtered for SMB2, and used File > Export Objects to pull
the actual CSV file straight out of the traffic. This is the file the departed employees grabbed.

![SMB file export in Wireshark](screenshots/smb_export.png)

## Part 2 — Pulling NTLM hashes out of the capture

Filtered for `ntlmssp` packets to get the Session Key, NTProofStr, and NT response tied to
each account, then saved them off so I wasn't hunting through packets again.

![NTLM auth details for both users](screenshots/ntlm_hashes.png)

## Part 3 — Cracking the password

Ran the NT hash through Hashcat against rockyou.txt:

```bash
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracked in under 30 seconds — not exactly a strong password.

![Hashcat cracking result](screenshots/hashcat_cracked.png)

## Part 4 — Verifying the NTLMv2 math by hand

Instead of trusting a tool blindly, I wrote a small script to recompute the HMAC-MD5 values
(NTProofStr and the Key Exchange Key) from the cracked password and compared them against
what was captured on the wire. They matched, which confirms the password belongs to that
exact session.

```bash
python3 hash.py --user mrealman --domain WORKGROUP --password <cracked_password> \
  --ntproofstr <captured_ntproofstr> --key <captured_session_key> -v
```

![Manual NTLMv2 hash validation](screenshots/hash_validation.png)

Same process for the second account:

```bash
python3 hash2.py --user eshellstrop --domain WORKGROUP --ntlmhash <ntlm_hash> \
  --ntproofstr <captured_ntproofstr> --key <captured_session_key>
```

![Second account hash validation](screenshots/hash_validation_2.png)

## Part 5 — Digging through the LSASS dump

Used pypykatz (pure Python, no need to touch a Mimikatz binary) to pull logon sessions,
NT hashes, and Kerberos keys straight out of memory for both accounts.

```bash
pip install pypykatz
pypykatz lsa minidump lsass.DMP | grep -i -C15 mrealman
pypykatz lsa minidump lsass.DMP | grep -i -C15 eshellstrop
```

![pypykatz output for first account](screenshots/pypykatz_mrealman.png)
![pypykatz output for second account](screenshots/pypykatz_eshellstrop.png)

## What this proves

Both former employees still had valid, active credentials after being terminated, and both
had logon sessions on the compromised host at the time the file was pulled off the server.
Between the packet capture and the memory dump, there's a clean chain from "credentials were
used" to "file was exfiltrated" — which is exactly what the investigation needed.

## Takeaway

Deactivate accounts the day someone leaves, not whenever someone gets around to it. This
whole incident traces back to one skipped step in offboarding.
EOF

echo "README written. Now review screenshots/ and rename files to match what's referenced above:"
echo "  smb_export.png, ntlm_hashes.png, hashcat_cracked.png, hash_validation.png,"
echo "  hash_validation_2.png, pypykatz_mrealman.png, pypykatz_eshellstrop.png"
echo ""
echo "IMPORTANT: crop or blur any real NT hashes / cracked passwords still visible in the images before pushing."
read -p "Press Enter once you've renamed/redacted the screenshots and are ready to push..."

git init
git add .
git commit -m "Add AD credential forensics lab writeup"
git branch -M main
git remote add origin "https://github.com/$GITHUB_USER/$REPO_NAME.git"

echo ""
echo "About to push to https://github.com/$GITHUB_USER/$REPO_NAME"
echo "Make sure you created this repo on GitHub first (empty, no README) and have a personal access token ready."
read -p "Press Enter to push, or Ctrl+C to stop here..."

git push -u origin main

echo "Done. Repo should be live at https://github.com/$GITHUB_USER/$REPO_NAME"
