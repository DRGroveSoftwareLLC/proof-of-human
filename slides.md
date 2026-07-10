---
marp: true
theme: uncover
class: invert
paginate: true
size: 16:9
footer: '![height:28px](assets/dr-grove-logo.png)'
style: |
  section {
      font-size: 2.5em !important;
  }
  footer {
    text-align: left;
  }
  footer img {
    filter: invert(1);
  }
  section.invert footer img {
    filter: none;
  }
  .compact {
    font-size: 0.75em !important;
  }
---

<!-- _class: lead invert -->

# Proof of Human

## Hardware-Backed, Cross-Signed OpenPGP Keys

Building a sovereign identity: from bare metal to Web of Trust

---

## What You'll Leave With Today

- A laptop-bootable **AirgapOS** thumb drive — ours, or one you built yourself
- An **SD card backup** of your key material
- A **hardware-backed OpenPGP identity**, generated entirely air-gapped
- At least one **peer signature** on your key — the actual "Proof of Human"

---

## Today's Agenda

1. Build & write AirgapOS to a thumb drive
2. Boot AirgapOS on a laptop
3. Create an advanced OpenPGP key with keyfork
4. Write the key to a hardware token
5. Wipe & reconstitute the hardware token
6. Upload your key to keys.openpgp.org
7. Sign another human's key

---

## Why "Proof of Human"?

- Software keys are copyable, stealable, silently exfiltrated
- Centralized identity (CAs, platforms) can be compelled, hacked, or revoked out from under you
- A **Web of Trust** — peer attestation instead of central authority (more shortly)
- Hardware-backed + cross-signed = an identity that is *yours*, and that other humans have verified is yours

---

<!-- _class: lead invert -->

# Concepts

---

## The Threat Model

- Malware on a networked machine can read a software private key in seconds
- Cloud key management means someone else is holding your key
- Phishing and social engineering don't work against a device that isn't listening
- **Goal:** private key material that never touches an online machine — ever

---

## What "Air-Gapped" Means

- No network hardware active, ever, during key operations
- AirgapOS strips wifi / bluetooth / audio drivers to shrink the exfiltration surface
- An air gap is a *physical* barrier between your seed and the internet, not just a logical one

---

## What "Hardware-Backed" Means

- Private key material is generated and stored on a smartcard or security token
- The key never leaves the token — signing happens *on* the device
- Malware on your daily driver can ask the token to sign something, but can't extract the key itself

---

## What "Cross-Signing" / Web of Trust Means

- A signature on someone's key is a public statement: *"I verified this human controls this key"*
- No central authority — trust is a graph you build one verified handshake at a time
- Your OpenPGP identity becomes more useful the more real humans vouch for it

---

<!-- _class: invert -->

## Glossary You'll Need Today

<div class="compact">

| Term | Meaning |
|---|---|
| Primary / Certify key (`C`) | The root of trust — only used to add subkeys and sign other keys |
| Subkeys (`S` `E` `A`) | Sign, Encrypt, Authenticate — your day-to-day operations |
| Fingerprint | The 40-hex-character unique ID of a key |
| UID | User ID — name / comment / email bound to a key |

</div>

---

## Don't Trust Us — Verify

- We're handing you a pre-built thumb drive + SD card today
- Don't want to trust our build? We'll show you how to build AirgapOS yourself, from source, and prove it's reproducible
- Same for keyfork — open source, and every command today is copy-pasteable straight from the public repo

---

<!-- _class: lead invert -->

# Step 1
## Build & Write AirgapOS

---

## What Is AirgapOS?

`git.distrust.co/public/airgap`

- Full-source-bootstrapped, deterministic, minimal, immutable
- **Diskless** — runs entirely from initramfs, nothing touches a disk
- Under 100MB; network drivers stripped out entirely

---

## What AirgapOS Is Built For

- Generating a PGP keychain
- Storing / restoring an OpenPGP keychain to a YubiKey or Nitrokey
- Signing cryptocurrency transactions
- Generating and backing up a BIP-39 wallet seed

---

## What You'll Need

- A build machine with **Docker 26+**
- A blank flash drive (the boot media)
- A blank SD card (your key-material backup, later)
- An x86_64 PC or laptop to boot it on

---

## Clone & Build

```sh
git clone https://git.distrust.co/public/airgap
cd airgap
git submodule update --init --recursive
make release
```

Output: `dist/airgap.iso` + `dist/manifest.txt`

<!-- Builds inside Docker with SOURCE_DATE_EPOCH=1 pinned — same inputs, same bytes, every time. This is what makes the reproducibility check on the next slide possible. -->

---

## Prove It's Reproducible

```sh
make reproduce
```

Match = provably the same build as what's in source control.

<!-- Rebuilds clean from scratch and diffs the new manifest against dist/manifest.txt. A match means no hidden step — what you built is what's in source control. -->

---

## Write It to Your Thumb Drive

```sh
lsblk                    # find your drive — do NOT guess
dd if=out/airgap.iso of=/dev/sdX bs=1M conv=sync status=progress
```

**`/dev/sdX` is your drive.** Get this wrong and you destroy data. Triple-check with `lsblk` first.

<!-- GUI alternative if anyone doesn't want the CLI: Balena Etcher or Rufus — same idea, more guardrails. -->

---

## Verify the Write

```sh
sha256sum out/airgap.iso
head -c $(stat -c '%s' out/airgap.iso) /dev/sdX | sha256sum
```

Hashes must match.

<!-- Confirms the ISO landed on the drive byte-for-byte. -->

---

<!-- _class: lead invert -->

# Step 2
## Boot AirgapOS on a Laptop

---

## Before You Boot

- Unplug ethernet, disable wifi/bluetooth if you can at the hardware level
- Disconnect any other storage you don't want visible to the session
- Insert your AirgapOS drive

---

## Getting Into the Boot Menu

| Vendor | Key |
|---|---|
| Dell / Acer / Lenovo | `F2` |
| HP | `F10` |
| ASUS / MSI | `Delete` |
| Some systems | `Esc` |

Watch the first splash screen — it usually tells you.

---

## Secure Boot Has to Go

AirgapOS is unsigned by design — it can't boot under Secure Boot.

1. Enter BIOS/UEFI Setup
2. Navigate to **Security / Boot / Authentication**
3. Set **Secure Boot** to **Disabled**
4. Save & Exit, reboot

---

## Boot AirgapOS

`Options -> Boot Options -> USB Boot`

Tested hardware: Purism Librem 14, HP 14" Celeron, Lenovo Flex 5i — any normal x86_64 laptop should work.

---

## Confirm You're Actually Air-Gapped

- Check for live network interfaces once booted
- No wifi / ethernet shown as up = you're isolated
- This is the environment everything from here happens in

---

<!-- _class: lead invert -->

# Step 3
## Create an Advanced OpenPGP Key with Keyfork

---

## What Is Keyfork?

`git.distrust.co/public/keyfork`

- One toolchain, one seed, many keys: OpenPGP, SSH, PIV, crypto-asset wallets
- Derives everything **deterministically** from a single BIP-39 mnemonic

---

## Why Determinism Matters

- Same seed in → same keys out, every time, forever
- Lose the hardware? Re-derive from the seed and get the *identical* key back
- This is the entire mechanism behind today's "wipe and reconstitute" step (Step 5)

---

## Key Architecture

- One **Certify (`C`)** key — the root; used only to add subkeys and to sign *other users' keys*
- **Sign (`S`)** subkey — signs arbitrary data (files, commits, messages) to prove it came from you
- **Encrypt (`E`) / Authenticate (`A`)** subkeys — the rest of what you use day to day
- Derived via Ed25519 (signing) and Curve25519 (encryption)

<!-- The Certify key's "sign other users' keys" capability is what Step 7 uses. -->

---

## Generating Entropy

```sh
keyfork mnemonic generate
```

Refuses to run unless the system is offline with an up-to-date kernel.

<!-- Override for testing only: INSECURE_HARDWARE_ALLOWED=1. Can also draw entropy from playing cards, tarot cards, or dice instead of the OS RNG. -->

---

## Protect the Mnemonic

- 12–24 words that **are** your identity — anyone with them can regenerate your exact key
- Write it on paper. Store it like cash or a passport.
- Never photograph it. Never type it into a networked device.

---

## The One-Liner

```sh
export KEYFORK_OPENPGP_EXPIRE=2y
keyfork mnemonic generate --encrypt-to-self encrypted.asc \
  --provision openpgp-card \
  --derive='openpgp --public "Your Name <your@email.co>"'
```

Mnemonic + derived identity + hardware token + encrypted seed backup — one command.

---

## Breaking Down the One-Liner

- `--encrypt-to-self` → seed encrypted to your new key as `encrypted.asc` (→ SD card)
- `--provision openpgp-card` → factory-resets the card, writes your subkeys to it (this *is* Step 4)
- `--derive` → UID string; always derives `C` + `S` + `E` + `A`

<!-- --encrypt-to-self: encrypts the newly generated seed to the OpenPGP key you just derived; encrypted.asc is what goes on the SD card backup.
--provision openpgp-card: factory-resets whatever smart card is plugged in and writes your derived Sign/Encrypt/Auth subkeys straight to it — this is Step 4, keyfork does it automatically.
--derive: UID format is "Full Name (optional comment) <email>" — any combination of name/username/email is valid; the openpgp format always derives C+S+E+A. -->

---

## The Manual, Step-by-Step Version

```sh
keyfork recover mnemonic
# once the agent is running, in the same session:
keyfork derive openpgp "Your Name <you@email.co>"
```

Same result as the one-liner.

<!-- recover mnemonic re-enters your mnemonic and starts a local keyfork agent; derive openpgp asks that agent for a fresh, deterministic OpenPGP cert. Useful for understanding what's actually happening underneath the one-liner. -->

---

## Review Your Key

Check that the UID, the fingerprint, and all four capabilities (`C`/`S`/`E`/`A`) are present and correct.

This is your identity from here forward.

---

## Back Up to Your SD Card

- Copy your mnemonic (on paper) **and** `encrypted.asc` to the SD card you were given
- If your hardware token is ever lost, stolen, or wiped — this backup is the only thing that matters

<!-- This is a different SD card purpose than the optional build-attestation flow from Step 1 — worth flagging if anyone did that step. -->

---

## Keys Expire on Purpose

- Default expiry is short — as little as **1 day** via `keyfork derive openpgp` directly
- We set `KEYFORK_OPENPGP_EXPIRE=2y` above — pick a value you'll actually remember to renew
- Short expiry forces a renewal habit instead of a "set and forget, then panic in 10 years" habit

---

<!-- _class: lead invert -->

# Step 4
## Write the Key to a Hardware Token

---

## Already Hardware-Backed

`--provision openpgp-card` in Step 3 already wrote your subkeys to the token — the "why" is back in Concepts.

---

## Which Devices?

Any device implementing the OpenPGP card applet:

**YubiKey · Nitrokey · OnlyKey** — whichever is in your kit today, the commands are identical.

---

## Confirm the Card

```sh
gpg --card-status
```

Shows the application ID, cardholder name, and the fingerprints of the keys now living on the token.

---

## Change the Default PINs — Now

- Factory defaults: **User PIN `123456`**, **Admin PIN `12345678`**
- keyfork's wizard prompts you to set real ones during provisioning — don't skip it
- 3 wrong **Admin PIN** attempts **destroys** the card
- 3 wrong **User PIN** attempts blocks it (Admin PIN can unblock)

---

<!-- _class: lead invert -->

# Step 5
## Wipe & Reconstitute the Hardware Token

---

## Why We Practice This Today

- Tokens get lost, stolen, dropped, run through the wash
- That's fine — **if** your seed backup is good
- We're going to prove it, on purpose, right now

---

## The Wipe

```sh
gpg --card-edit
gpg/card> admin
gpg/card> factory-reset
```

Destroys all key material on the card and resets both PINs to factory defaults.

---

## Confirm It's Actually Wiped

```sh
gpg --card-status
```

Should show an empty card — no keys, no cardholder name.

---

## The Reconstitute

```sh
keyfork recover mnemonic
# re-enter the same mnemonic from your SD card backup
keyfork derive openpgp "Your Name <you@email.co>"
```

Re-derives the exact same OpenPGP certificate from the same seed.

---

## Why the Fingerprint Comes Back Identical

Same seed → **bit-for-bit identical key, every time.**

<!-- keyfork derives OpenPGP keys with a fixed creation timestamp (Unix epoch + 1), so identical seed + identical timestamp always produces an identical key. Verify by comparing the gpg --card-status fingerprint to the original. -->

---

## The Takeaway

**The token is disposable. The seed is sovereign.**

Protect the backup, not the token.

---

<!-- _class: lead invert -->

# Step 6
## Upload to keys.openpgp.org

---

## What a Keyserver Does (and Doesn't Do)

- Distribution — a place people can find your public key
- **Not** trust — anyone can upload a key claiming to be anyone
- Trust comes from Step 7: verified human signatures, not the server

---

## keys.openpgp.org Specifics

- Hagrid-based: **email-verified** UIDs only
- Strips **third-party certifications** on upload — your signature on someone else's key won't survive an upload to this server

<!-- Keep that in mind for Step 7 — it's why you'll send signatures directly instead of uploading them. -->

---

## Export Your Public Key

```sh
gpg --export --armor your@email.co > mykey.asc
```

---

## Upload

```sh
gpg --send-keys --keyserver hkps://keys.openpgp.org YOUR_FINGERPRINT
```

Or use the web form at `keys.openpgp.org/upload` and paste `mykey.asc`.

---

## Verify Your Email

- keys.openpgp.org emails a confirmation link to each UID's address
- Until you click it, your UID isn't searchable — only the bare key is
- Check your inbox now

---

## Confirm It's Live

```sh
gpg --locate-keys your@email.co
```

Or search `https://keys.openpgp.org` directly by email or fingerprint.

---

<!-- _class: lead invert -->

# Step 7
## Sign Another Human's Key

---

## Why This Step *Is* the Event

Everything so far built **your** identity.

This is the step where two humans, standing in the same room, turn that into a **verified** identity — the actual Proof of Human.

---

## In-Person Verification Protocol

1. Check a government ID against the name on their key
2. Have them read their fingerprint aloud
3. Don't trust a screen — trust what was said out loud and the ID in your hand

---

## Exchange Fingerprints Out-of-Band

- Paper slip, or spoken aloud — **not** screen-to-screen
- You already confirmed this laptop is air-gapped; don't casually undo that by trusting arbitrary display output for identity data

---

## Import Their Public Key

```sh
gpg --recv-keys --keyserver hkps://keys.openpgp.org THEIR_FINGERPRINT
```

Or import directly from a file/QR code they hand you.

---

## Verify the Fingerprint — Character by Character

```sh
gpg --fingerprint THEIR_FINGERPRINT
```

Compare every group of characters against what they read aloud. **All of it. Every time.**

---

## Sign

```sh
gpg --sign-key THEIR_FINGERPRINT
```

Ideally performed from your Certify key, offline — the same posture as key generation.

---

## Export & Send It Back

```sh
gpg --export THEIR_FINGERPRINT | gpg --encrypt --armor -r THEIR_FINGERPRINT > sig-for-them.asc
```

keys.openpgp.org strips third-party certs — so you send your signature directly, encrypted to them.

<!-- They import it and can choose to publish/attach it themselves. -->

---

## What This Builds

A real, small, high-trust **Web of Trust** edge — not a metaphor, an actual cryptographic statement that two humans verified each other today.

Do this again. Every signature you collect makes the graph stronger.

---

<!-- _class: lead invert -->

# Wrap-Up

---

## Recap

Today you built:

- A sovereign OpenPGP identity, generated fully air-gapped
- A hardware-backed key that survived being wiped and reconstituted
- A backup that means the hardware was never the point
- At least one real, human-verified signature

---

## When You Get Home

- Verify your AirgapOS build's reproducibility: `make reproduce`
- Collect more signatures — conferences, meetups, coworkers
- Set a calendar reminder for your key's expiration
- Store the SD card backup somewhere as secure as your mnemonic deserves

---

## Further Reading

- `gpg.wtf` — plain-language OpenPGP notes and Web of Trust mechanics
- `book.hashbang.sh` — hardware security modules, key management, hardening
- `git.distrust.co/public/airgap`
- `git.distrust.co/public/keyfork`

---

## Thanks

Questions?

---

<!-- _class: lead invert -->

## Sponsors

**Localhost Research**

**Nitrokey**

**DR Grove Software LLC**
