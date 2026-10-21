# Lab 2A — Crack an unsalted MD5 dump

**Goal:** crack a copy of weakly-hashed passwords yourselves, and see why "just hash it" is not
enough. **About 15 minutes.** Standalone — no Docker needed for this part.

!!! warning "Ethics — only your own dump"
    This attack runs **only** on a dump you generate yourself, from the course's own seed data. The
    technique is identical to what you would run against a real leaked database — that is exactly
    why it only ever touches targets you are allowed to touch.

!!! note "This is a race"
    Work in your pair. First pair to crack all 6 accounts shouts it out. No penalty for finishing
    second — the goal is energy, not pressure — but it is genuinely satisfying to win one.

Right now, NotesApp stores passwords in **plain text** — not even hashed (that is flaw #1, fixed in
[Lab 2B](password-fix.md)). This part is a separate, standalone exercise: imagine a *different* app
that at least thought to hash its passwords, but did it badly — MD5, no salt. You will see that
"badly hashed" is barely better than not hashed at all.

## Step 1 — Build the leaked dump

Make the `crack/` folder first:

```bash
mkdir -p crack
```

Create `crack/make_dump.py` — it reads the same usernames and passwords seeded in `db/init.sql` and
hashes each with plain MD5, no salt:

```python
import hashlib

# Same accounts NotesApp seeds today (db/init.sql) - copied here only for this offline exercise.
accounts = {
    "admin": "admin123",
    "prof.turing": "teach123",
    "alice": "alicepw",
    "bob": "bobpw",
    "carol": "carolpw",
    "mallory": "mallorypw",
}

with open("crack/leaked_users.txt", "w") as f:
    for username, password in accounts.items():
        digest = hashlib.md5(password.encode()).hexdigest()
        f.write(f"{username}:{digest}\n")

print("wrote crack/leaked_users.txt")
```

Run it:

```bash
python3 crack/make_dump.py
cat crack/leaked_users.txt
```

!!! tip "Windows: `python3` not found?"
    On Windows, use `python` instead of `python3` for every command in this lab (and `type` instead
    of `cat` to print a file). If neither works, Python is not on your PATH — reopen your terminal
    after installing Python, or use the "Add to PATH" option in the Python installer.

!!! success "Expected output"
    Six lines, each `username:32-character-hex-string`, e.g.:

    ```
    admin:0192023a7bbd73250516f069df18b500
    prof.turing:...
    alice:...
    ```

This is exactly what an attacker sees after stealing the table of a site that hashes with unsalted
MD5. Notice: no salt means the digest depends only on the password. If two users ever picked the
same password, their hashes would be **identical** — a real-world tell that lets attackers crack
many accounts at once.

## Step 2 — Build a cracking wordlist

Real attackers use huge leaked-password lists (`rockyou.txt` has 14 million entries). We use a short
one so this finishes in seconds. Create `crack/wordlist.txt`:

```
password
password123
123456
qwerty
letmein123
admin123
teach123
alicepw
bobpw
carolpw
davepw
mallorypw
dragon123
sunshine1
```

Notice the list mixes two kinds of guesses: generic top-of-the-charts passwords (`password123`,
`qwerty`), and guesses **targeted at this app** — `<username>pw` — built because an attacker who
sees `alicepw` leak once can guess the whole site follows the same pattern. This combination
(generic list + pattern guesses) is how real attackers build wordlists.

## Step 3 — Crack it

Create `crack/crack.py`:

```python
import hashlib

def load_dump(path):
    accounts = {}
    with open(path) as f:
        for line in f:
            username, digest = line.strip().split(":")
            accounts[username] = digest
    return accounts

def load_wordlist(path):
    with open(path) as f:
        return [w.strip() for w in f if w.strip()]

accounts = load_dump("crack/leaked_users.txt")
wordlist = load_wordlist("crack/wordlist.txt")

cracked = {}
for candidate in wordlist:
    candidate_digest = hashlib.md5(candidate.encode()).hexdigest()
    for username, digest in accounts.items():
        if username in cracked:
            continue
        if digest == candidate_digest:
            cracked[username] = candidate

print(f"cracked {len(cracked)} / {len(accounts)} accounts:")
for username, password in cracked.items():
    print(f"  {username} : {password}")
```

Run it:

```bash
python3 crack/crack.py
```

!!! success "Expected output"
    ```
    cracked 6 / 6 accounts:
      admin : admin123
      prof.turing : teach123
      alice : alicepw
      bob : bobpw
      carol : carolpw
      mallory : mallorypw
    ```

!!! success "Checkpoint"
    All six accounts cracked, in well under a second, with a 14-entry wordlist and a plain laptop. No
    GPU, no special tool — this is `hashlib`, built into Python.

Move straight to [Lab 2B](password-fix.md), or the bonus below if you have time left. The "why was
that so fast" discussion happens together as a class on the debrief slide, not here — don't wait on
it.

## Bonus — crack a 7th account the wordlist never contained (optional)

??? tip "Rule-based cracking"
    Add two more accounts to `accounts` in `crack/make_dump.py` and re-run it:

    ```python
    "eve": "evepw",
    "frank": "frankpw",
    ```

    Run `crack.py` again as-is. Notice `eve` and `frank` **stay uncracked** — their passwords are not
    literal entries in `wordlist.txt`.

    Now fix `crack.py` itself instead of growing the wordlist by hand: after loading `wordlist`, also
    generate one **rule-based** candidate per username — `username + "pw"` — and test those too:

    ```python
    rule_based = [f"{u}pw" for u in accounts]
    candidates = wordlist + rule_based
    ```

    (swap your `for candidate in wordlist:` loop to loop over `candidates` instead). Re-run — now all
    8 accounts crack, including the two you just added.

    This is the real difference between a plain dictionary attack and a **rule-based** attack:
    instead of only trying known passwords, you generate new guesses from patterns you noticed (here:
    "this app seems to use `username + pw`"). Tools like Hashcat and John the Ripper build entire
    rule languages around exactly this idea.
