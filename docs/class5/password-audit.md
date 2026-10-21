# Lab 2 — Audit: how NotesApp stores passwords

**Your job:** audit how NotesApp stores passwords. Prove the current storage is weak by breaking it,
fix it, and **measure how much the fix actually buys** — in real numbers you record yourself.
About 35–40 minutes, with your pair.

!!! abstract "What you hand in — a findings sheet, not code"
    The deliverable is the **Audit summary** at the bottom: two findings and one measurement table,
    all short fills. Use AI for the code if you want — everyone will. AI cannot run *your* clock,
    read *your* capture, or make *your* severity call. That judgment is the graded part, and it is
    the part that is actually yours.

!!! warning "Ethics — only your own data"
    Every attack here runs **only** on a dump you generate yourself, from the course's own seed data,
    or on your own NotesApp clone. Same technique as a real leaked database — which is exactly why it
    only ever touches targets you are allowed to touch.

---

## Finding 1 — How fast can an attacker read the stored passwords?

NotesApp today stores passwords in **plain text** — already the worst case. So instead, let's test
the defense that *feels* safe and that many real apps shipped: "we hashed them" — with MD5, no salt.
You are going to measure how strong that actually is.

!!! question "Predict first, with your pair"
    You are handed a leaked table of 6 accounts, hashed with unsalted MD5. You attack it with a
    **real public wordlist** — the kind that actually circulates, not something we tailored to these
    accounts. Before you run it: **how many of the 6 fall, and how long does it take?** Will *all* of
    them? Write your guess down.

??? note "How to build and run the attack (scaffolding — use it, or ask AI to)"
    This is the mechanical part. Copy it, or have AI generate it — the point is the result, not the
    typing. The wordlist is a real one, **not** written around these passwords: some accounts will
    fall, some will not. That is the honest version of this attack.

    Make the folder and get the wordlist:

    ```bash
    mkdir -p crack
    ```

    **⬇ Download the wordlist** (≈190 of the world's most common passwords, a small slice of what
    attackers really use) and save it as `crack/wordlist.txt`:
    [`common-passwords.txt`](assets/common-passwords.txt)

    `crack/make_dump.py` — a leaked table from "some other app", with a realistic spread of password
    strength. We hash each with unsalted MD5, exactly as that app would have:

    ```python
    import hashlib

    accounts = {
        "liwei": "123456",
        "sofia": "iloveyou",
        "tomk":  "qwerty123",
        "aisha": "password1",
        "marie": "mariepw",        # not a dictionary word - a predictable pattern
        "raj":   "Gv7#pLm2!xQ",    # long and random
    }
    with open("crack/leaked_users.txt", "w") as f:
        for username, password in accounts.items():
            f.write(f"{username}:{hashlib.md5(password.encode()).hexdigest()}\n")
    print("wrote crack/leaked_users.txt")
    ```

    `crack/crack.py` — tries every word in the real wordlist, and times itself:

    ```python
    import hashlib, time

    def load_dump(path):
        out = {}
        with open(path) as f:
            for line in f:
                u, d = line.strip().split(":")
                out[u] = d
        return out

    accounts = load_dump("crack/leaked_users.txt")
    wordlist = [w.strip() for w in open("crack/wordlist.txt") if w.strip()]

    start = time.time()
    cracked = {}
    for candidate in wordlist:
        cd = hashlib.md5(candidate.encode()).hexdigest()
        for u, d in accounts.items():
            if u not in cracked and d == cd:
                cracked[u] = candidate
    elapsed = time.time() - start

    print(f"cracked {len(cracked)} / {len(accounts)} in {elapsed*1000:.1f} ms")
    for u, p in cracked.items():
        print(f"  {u} : {p}")
    ```

    Run it: `python3 crack/make_dump.py && python3 crack/crack.py`
    (Windows: use `python` instead of `python3`.)

!!! success "📏 Measure — record these three things"
    Run the attack and read the output:

    - Accounts cracked: **___ / 6**
    - Time taken: **___ ms**
    - **Which accounts did NOT crack?** ___________

    Then check your prediction. (Expect **4 of 6** — the genuinely common passwords — in a few
    milliseconds. Two survive.)

!!! question "Why did those two survive? (discuss with your pair)"
    You know all 6 passwords — they are in `make_dump.py`. Look at the two that did not crack.

    ??? success "Answer"
        `marie` used `mariepw` — not a dictionary word, so it is not in the public list (yet — see the
        bonus). `raj` used a long random password that no wordlist contains. **Cracking is not
        magic.** It feeds on weak and predictable passwords. A fast, unsalted hash just makes each
        guess instant — it does not invent passwords the attacker never guesses.

**Write Finding 1** (one line each — this goes in your Audit summary):

```
FINDING 1 — stored passwords (unsalted MD5)
  Exposed:         ___ of 6 accounts, in ___ ms, with a public wordlist
  Why those fell:  ________________________________
  Which survived & why: ___________________________
  Severity:        (Low / Med / High) because _______
  Fix (1 line):    ________________________________
```

---

## Remediation — migrate NotesApp to Argon2

Only `raj`'s random password survived — and you cannot ship security that assumes every user is
`raj`. The real fix protects the `liwei`s and `marie`s too, by making **every single guess
expensive**, strong or weak. That is Argon2.

Now fix it on the real app. You are adding Argon2 **and** upgrading each account the moment its owner
next logs in — no migration script, no downtime. NotesApp should be running (`docker compose up -d`).

??? note "How to apply the fix (scaffolding — three small changes)"
    Add the dependency — edit `web/requirements.txt`, add `argon2-cffi==23.1.0`, then
    `docker compose build web`.

    In `web/app.py`, near the imports:

    ```python
    from argon2 import PasswordHasher
    from argon2.exceptions import VerifyMismatchError, InvalidHash
    ```

    Just below `app = Flask(__name__)`:

    ```python
    ph = PasswordHasher()

    def is_argon2_hash(value: str) -> bool:
        return value.startswith("$argon2")
    ```

    In the `login()` view, **three changes**:

    1. In the query, add `, password` to the SELECT and drop `AND password = '...'` from the WHERE
       (leave `username` concatenated — the SQL-injection flaw stays, it is December's class).
    2. Insert this block just before `conn.close()`:

        ```python
        authenticated = False
        if row:
            stored = row["password"]
            if is_argon2_hash(stored):
                try:
                    ph.verify(stored, password)
                    authenticated = True
                except (VerifyMismatchError, InvalidHash):
                    authenticated = False
            else:
                if stored == password:
                    authenticated = True
                    new_hash = ph.hash(password)
                    conn.cursor().execute(
                        "UPDATE users SET password = %s WHERE id = %s",
                        (new_hash, row["id"]),
                    )
                    log(f"upgraded {row['username']} to argon2")
        ```

    3. Change the `if row:` line (after `conn.close()`) to `if authenticated:`. Everything below it
       stays as-is.

    Rebuild: `docker compose up -d --build`.

!!! danger "Whatever writes the code — you must understand the `else` branch"
    That `else` branch is the whole idea: an old plain-text account is checked once, then silently
    re-hashed with Argon2 on that login. If AI wrote it for you, read it until you can explain it —
    the next prediction depends on you getting this.

!!! question "Predict — before you look at the database"
    Log in once as `alice` / `alicepw` (normal login, nothing special). Then, **before** you query
    the DB: what does `alice`'s stored `password` hold now — still `alicepw`, or something else? Why?

??? success "Check it, and were you right?"
    ```bash
    docker compose exec db psql -U notesapp -d notesapp -c "SELECT username, password FROM users WHERE username = 'alice';"
    ```

    It now shows `$argon2id$v=19$m=65536,t=3,p=4$...` — a long Argon2 hash, not `alicepw`. Her one
    normal login hit the `else` branch, which re-hashed and wrote it back. No reset, no downtime —
    this is how real systems migrate password storage on live users.

!!! success "Checkpoint — the app is now in the right state"
    `alice` is migrated. Log out and back in once more — it still works, now via the `is_argon2_hash`
    branch. The app ends this lab with Argon2 storage. Leave it running for the next measurement.

---

## Finding 2 — How much did the fix actually buy? (measure)

Same attacker, same laptop. The only thing that changed is the hash algorithm's **speed**. Measure
the difference yourself.

!!! question "Predict first"
    MD5 just cracked 6 accounts in milliseconds. Argon2 is built to be *slow on purpose*. Guess the
    ratio: Argon2 is **how many times** slower per guess than MD5? 10×? 1000×? More?

??? note "How to measure (one command — runs inside the web container, which has Argon2)"
    ```bash
    docker compose exec web python - <<'PY'
    import hashlib, time
    from argon2 import PasswordHasher

    N = 100_000
    t = time.time()
    for _ in range(N):
        hashlib.md5(b"guess").hexdigest()
    md5_rate = N / (time.time() - t)
    print(f"MD5    : {md5_rate:,.0f} guesses/second")

    ph = PasswordHasher()
    t = time.time()
    ph.hash("guess")
    one = time.time() - t
    print(f"Argon2 : {one*1000:.0f} ms for ONE guess  ({1/one:.1f} guesses/second)")
    PY
    ```

!!! success "📏 Measure — fill the table"
    Read the two output lines, then compute the last column (`guesses/second × 86 400`):

    | Hash | Guesses / second | Guesses an attacker makes in **one day** |
    |---|---|---|
    | MD5 | ____________ | ____________ |
    | Argon2 | ____________ | ____________ |

**Write Finding 2 + the verdict** (goes in your Audit summary):

```
FINDING 2 — the fix, measured
  Argon2 is about ______ times slower per guess than MD5.
  Verdict — with your stolen database, which hash does the
  attacker beat, which do they give up on, and WHY (one sentence):
  ____________________________________________________________
```

!!! note "Why this isn't the same as re-running the crack"
    You are measuring *raw guessing speed*, which is what actually decides whether a real attacker
    (with a wordlist of millions, not ~190) wins. The wordlist is the same for both hashes — only the
    per-guess cost changed, and that is the entire story of Argon2.

---

## Audit summary — hand this in

Copy your fills into one place:

```
NOTESAPP PASSWORD AUDIT — pair: ____________

FINDING 1 — stored passwords (unsalted MD5)
  Exposed:      ___/6 cracked in ___ ms, public wordlist
  Survived & why: ____
  Severity:     ___ because ____
  Fix:          ____

FINDING 2 — the fix, measured
  MD5:    ____ guesses/sec   → ____ / day
  Argon2: ____ guesses/sec   → ____ / day
  Argon2 is ~____× slower per guess.
  Verdict: ____

STATE: NotesApp now stores passwords with Argon2 (alice migrated on login).  ✅
```

??? tip "Finished early? (optional) — crack `marie` with a guessing rule"
    `marie`'s password (`mariepw`) is not in any public wordlist, so the dictionary attack missed it.
    But it follows a **pattern** — `<username>pw` — and an attacker who spots a pattern can generate
    guesses from it. Add rule-based candidates and test them too:

    ```python
    rule_based = [f"{u}pw" for u in accounts]
    candidates = wordlist + rule_based
    ```

    (loop over `candidates` instead of `wordlist`). Now `marie` falls — **5 of 6**. Only `raj`, the
    long random password, still survives. That is the jump from a plain dictionary attack to a
    **rule-based** one — Hashcat and John the Ripper are built around exactly this. Add a note to your
    audit: "the wordlist size is not the limit; attacker creativity is — the only thing that held was
    a password with no pattern at all."

## Troubleshooting

??? bug "`ModuleNotFoundError: No module named 'argon2'`"
    You edited `requirements.txt` but did not rebuild. Run `docker compose build web`, then
    `docker compose up -d`.

??? bug "Login always fails after the fix"
    Indentation — the `if authenticated:` block must sit exactly where the old `if row:` was. Check
    the container logs: `docker compose logs web` (an `IndentationError` confirms it).

??? bug "DB still shows the plain password after login"
    Confirm you are hitting the rebuilt container: `docker compose ps` should show `web` with a
    recent `CREATED` time.
