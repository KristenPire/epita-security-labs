# Lab 2B — Fix it with Argon2

**Goal:** fix NotesApp's real login so passwords are never stored in clear text again — migrate to
Argon2, upgrading each account the moment its owner logs in. **About 20–25 minutes.**

NotesApp should be running (`docker compose up -d`). Work inside your `notesapp/` clone.

Today, `notesapp/db/init.sql` stores every password in plain text, and `notesapp/web/app.py`
compares it directly in the SQL query. We will:

1. Add Argon2 hashing to the login check.
2. **Upgrade accounts automatically**, the moment their owner logs in — no separate migration
   script, no downtime, no forcing 30 students to reset their password today.

## Step 1 — Add the dependency

Edit `notesapp/web/requirements.txt`, add one line:

```
argon2-cffi==23.1.0
```

Rebuild so the new dependency installs:

```bash
docker compose build web
```

## Step 2 — Add the hashing helpers

Open `notesapp/web/app.py`. Near the top, with the other imports, add:

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError, InvalidHash
```

Just below where `app = Flask(__name__)` is defined, add:

```python
ph = PasswordHasher()  # Argon2id, sane defaults - no tuning needed for this course

def is_argon2_hash(value: str) -> bool:
    return value.startswith("$argon2")
```

## Step 3 — Change the login check

Find the `login()` view (`@app.route("/login", ...)`).

!!! danger "Leave the SQL injection alone — on purpose"
    **Leave the string-concatenation style of the query exactly as it is for the username.** That SQL
    injection bug is planted on purpose and is fixed in the December injection class, not today — if
    you parameterize it now, you remove a flaw the course still needs. Today we only change how the
    **password** is checked, which happens to mean the password itself can no longer sit inside the
    SQL string, since it is no longer compared with `=`.

Do **not** re-type the whole view. You are making **three small changes** to the `login()` code you
already have. Make them in order.

### Change 1 — fetch the stored password, stop letting SQL check it

The query today checks the password inside the SQL (`AND password = '...'`). We want to fetch the
stored password and check it ourselves in Python instead. Edit the two string lines:

```python
        query = (
            "SELECT id, username, role, password FROM users "   # (1) add  , password
            "WHERE username = '" + username + "'"               # (2) drop  AND password = ...
        )
```

Only those two lines change. Leave `username` concatenated as-is — that is the planted SQL injection,
left for December.

### Change 2 — verify (and upgrade) the password in Python

Right now, straight after the `try/except` that sets `row`, the code does `conn.close()` then
`if row:`. **Insert this block between them** — just before the `conn.close()` line:

```python
        authenticated = False
        if row:
            stored = row["password"]
            if is_argon2_hash(stored):
                # Already migrated: check with Argon2.
                try:
                    ph.verify(stored, password)
                    authenticated = True
                except (VerifyMismatchError, InvalidHash):
                    authenticated = False
            else:
                # Legacy plain-text account: check it, then upgrade it on the spot.
                if stored == password:
                    authenticated = True
                    new_hash = ph.hash(password)
                    conn.cursor().execute(
                        "UPDATE users SET password = %s WHERE id = %s",
                        (new_hash, row["id"]),
                    )
                    log(f"upgraded {row['username']} to argon2")
```

Read it before you run it — this block **is** the lesson. Two cases: an already-Argon2 account is
verified with `ph.verify`; a legacy plain-text account is checked, and if it matches, re-hashed and
written back right away, so it is migrated the moment its owner logs in.

### Change 3 — gate the login on `authenticated`, not `row`

Find the `if row:` line (just after `conn.close()`) and change it to:

```python
        if authenticated:
```

Everything below it (`session["user_id"] = ...` and so on) stays exactly as it was.

!!! note "Why the UPDATE uses %s but the username lookup does not"
    The `UPDATE` uses `%s` placeholders (psycopg2 parameter substitution) — the correct, safe way to
    run a query with a value you do not fully control. We use it here because this one genuinely is
    **not** the planted flaw; only the username lookup is (and that stays, for December).

## Step 4 — Rebuild and test

```bash
docker compose up -d --build
```

**Before testing, look at the database directly:**

```bash
docker compose exec db psql -U notesapp -d notesapp -c "SELECT username, password FROM users WHERE username = 'alice';"
```

!!! success "Expected output"
    The `password` column still shows `alicepw`, in clear text — nothing has run yet.

Now log in as `alice` / `alicepw` at <http://localhost:8080> (or `https://localhost` if you kept
Lab 1's HTTPS setup running).

!!! success "Checkpoint"
    The login succeeds, same as before.

Check the database again:

```bash
docker compose exec db psql -U notesapp -d notesapp -c "SELECT username, password FROM users WHERE username = 'alice';"
```

!!! success "Expected output now"
    The `password` column shows something like `$argon2id$v=19$m=65536,t=3,p=4$...` — a long Argon2
    hash, not `alicepw` anymore.

Log out, log back in as `alice` / `alicepw` one more time.

!!! success "Checkpoint — you're done"
    Still works — this time it went through the `is_argon2_hash` branch (`ph.verify`), not the legacy
    branch. The account is now fully migrated. That's the lab, once this checkpoint passes.

!!! tip "Optional, if you have time"
    Log in as `alice` with the **wrong** password. Still rejected, same "Invalid credentials" message
    as before the change — an attacker watching the login form cannot tell a migrated account from a
    legacy one just by the error message.

!!! note "Why we don't re-crack the new Argon2 hash here"
    Copy-pasting a hash value by hand is slow and typo-prone for not much payoff, so just reason about
    the result: in Lab 2A, the whole wordlist cracked all 6 MD5 accounts in **under one second**.
    Argon2 is deliberately slow — roughly **0.1 second per single guess** on a laptop. Same wordlist,
    same attacker: against Argon2 it goes from under a second to **hours**, purely because of the hash
    algorithm's speed. That ~million-times slowdown, with nothing else changed, is the whole point of
    Argon2. You will see this compared side by side on the debrief slide.

## Wrap-up checklist (confirm before leaving)

- [ ] `crack/leaked_users.txt` created, all 6 accounts cracked from `wordlist.txt`.
- [ ] `notesapp/web/app.py` `login()` now checks Argon2 hashes, with legacy plain-text fallback that
      upgrades on successful login.
- [ ] Logged in as `alice` twice: first login upgraded the account, second login used the Argon2
      branch.
- [ ] Verified the SQL injection on `username` is still present (do **not** parameterize it today —
      that is December's class).
- [ ] Can state in one sentence why Argon2 resisted the same attack that broke MD5 in seconds.

## Troubleshooting

??? bug "ModuleNotFoundError: No module named 'argon2'"
    You edited `requirements.txt` but did not rebuild. Run `docker compose build web` again, then
    `docker compose up -d`.

??? bug "Login always fails after your edit"
    Check indentation carefully; the `if authenticated:` block must be indented exactly like the old
    `if row:` block was. A Python `IndentationError` in the container logs (`docker compose logs
    web`) means this is the issue.

??? bug "The database still shows the plain password after login"
    Confirm you are hitting the container you rebuilt: `docker compose ps` should show `web` with a
    recent `CREATED` time.
