# Project 2 — Password manager

Build a small **command-line password manager** in Python. Everything it needs, you already built
or saw in class: Argon2 (Class 5, today), AES-GCM (Class 3), key derivation (Argon2 again),
asymmetric encryption (Class 4).

- **Groups of 2–3.**
- **Starts 22 Oct · due 12 Nov** (hand in by end of day).
- **Worth 25% of your course grade.**

!!! abstract "The deliverable is a working tool **and** a README that justifies every choice"
    You may use AI to write code — you will anyway, and that is fine. So the code is *half* the
    grade. The other half is the **README**, where you explain **why** each cryptographic building
    block is the right one — not that it runs, but why it is correct. That is the part AI can't hand
    you: you have to understand your own tool well enough to defend it.

## What it does (minimum)

A CLI that lets a user:

- set a **master password** once (first run),
- unlock the vault with that master password,
- **add / list / get / delete** entries (a site, a username, a password),
- **generate** a strong password on demand.

A simple session is enough — no GUI, no network (except the optional breach check). Example:

```text
$ python vault.py init
$ python vault.py add github --user alice
$ python vault.py get github
$ python vault.py gen --length 20
```

(Exact commands are your design — these are just a shape.)

## Required — four building blocks

Each one maps to something you have already done. In your README, name the library you used and why.

!!! note "1 · Master password stored with Argon2"
    The master password is **hashed with Argon2** (never stored in clear text, never recoverable).
    On unlock, you **verify** the entered password against the stored hash.

    - *From:* Class 5 lab — you wrote this exact code.
    - *Done when:* the stored master-password value is an `$argon2id$...` string, and a wrong master
      password is rejected.

!!! note "2 · Encryption key **derived** from the master password"
    The key that encrypts the vault is **derived from the master password** (Argon2 as a key
    derivation function), not stored anywhere on disk.

    - *Done when:* there is **no key file**. The only secret that unlocks the vault is the master
      password in the user's head.
    - *Design question you must answer in the README:* why derive the key from the master password
      instead of generating a random key and saving it in a file next to the vault?

!!! note "3 · Vault encrypted with AES-GCM"
    The vault (all the stored entries) is encrypted with **AES-GCM** — authenticated encryption, so
    any tampering with the vault file is **detected** on decrypt, not silently accepted.

    - *From:* Class 3.
    - *Done when:* flipping a single byte of the vault file makes decryption **fail loudly**, not
      return garbage.

!!! note "4 · Strong password generator"
    Generate strong passwords using a **cryptographically secure** random source (Python's `secrets`,
    not `random`). Let the user pick a length.

    - *Done when:* passwords are long, random, and use a wide character set — and you can explain in
      the README why `secrets` and not `random`.

!!! warning "Rule: do not roll your own crypto"
    Use established libraries — `argon2-cffi` for Argon2, `cryptography` for AES-GCM and key
    derivation. Inventing your own encryption or hashing is an automatic fail, exactly as in real
    engineering. The skill is choosing and combining the right primitives, not reinventing them.

## The security model (state this in your README)

Say clearly **what your tool protects against and what it does not**:

- It **protects** the stored passwords if someone steals the vault file — they get only AES-GCM
  ciphertext, unlockable only with the master password (which never touches disk).
- It does **not** protect against: a keylogger on the user's machine, a weak master password, or the
  user's screen being read. Name at least these.

A tool whose author knows its limits is worth more than one whose author thinks it is magic.

## Bonus (optional — only after the four required blocks work)

!!! tip "A · Share one entry with asymmetric encryption"
    Let a user share **one** vault entry with a teammate by encrypting it with the teammate's
    **public key**, so only their **private key** can open it. *From:* Class 4.

!!! tip "B · Breach check with k-anonymity (Have I Been Pwned)"
    Check whether a password has appeared in a known breach, using the **Have I Been Pwned** range
    API — which uses **k-anonymity**: you send only the first 5 characters of the password's SHA-1
    hash, never the password or its full hash. Explain in the README why that is safe to send.

## The README — where the grade is

Short, but it must answer **why**, for each choice:

1. Why **Argon2** for the master password, and not SHA-256 or AES? (Think back to today's lab.)
2. Why **derive** the encryption key from the master password instead of storing a key file?
3. Why **AES-GCM** and not AES without authentication? What does the "GCM" part buy you?
4. Why **`secrets`** and not `random` for generating passwords?
5. What your tool does **not** protect against.

"It works" is not an answer to any of these. "It is the right choice because…" is.

## How it's graded (25% of the course)

| Part | Weight | What we look for |
|---|---|---|
| The four required blocks work | 40% | Each "done when" check passes |
| README justifies every choice | 40% | The *why*, correct and in your own words — the AI-proof half |
| Security model stated honestly | 10% | You know your tool's limits |
| Code quality + a bonus attempted | 10% | Readable; one bonus is plenty |

!!! danger "You will be asked to explain your own code"
    Be ready to answer, as a group, why any line of your crypto does what it does. A tool you can't
    explain is a tool you don't own — and that is exactly what this project grades.

## Getting started

- Form your group (2–3) and pick who owns the vault format vs the CLI vs the README — then swap, so
  everyone touches the crypto.
- Reuse your own Class 5 Argon2 code and your Class 3 AES-GCM code as starting points.
- Install what you need: `pip install argon2-cffi cryptography`.
- Due **12 November** — code + README in one repository.
