# Lab — Fix the login: make the forged token useless

**Your job:** close the hole you opened last class. You forged a teacher token two ways — one with
no signature (`alg:none`), one signed with the cracked secret `s3cr3t`. Today you change NotesApp
so **both bounce**, without breaking normal login. About 35 minutes, with your pair.

Work inside your `notesapp/` clone. Bring your **two forged tokens from last class** — you test
against both.

!!! abstract "What you hand in — the fixed function + one check"
    The deliverable is your fixed `verify_jwt`, plus the **Audit line** at the bottom proving both
    forged tokens now return `unauthorized` and a real login still works.

!!! warning "Only your own app"
    You attack and fix **only** your own NotesApp clone, on your own machine.

!!! question "Predict first — commit with your pair"
    After all three fixes, **will your old forged tokens still work?** And will a normal login still
    work? Write yes / no for each, before you start.

---

## Start the app and find the code

```bash
docker compose up -d
```

Open `web/app.py`. Find two things:

1. The secret, near the top:

    ```python
    JWT_SECRET = os.environ.get("JWT_SECRET", "s3cr3t")
    ```

2. The checker, `verify_jwt(...)` — the broken version:

    ```python
    alg = header.get("alg", "").lower()
    if alg == "none":
        # No signature verification at all.
        return payload
    if alg == "hs256":
        signing_input = f"{h_b64}.{p_b64}".encode()
        expected = _b64url(hmac.new(JWT_SECRET.encode(), signing_input, hashlib.sha256).digest())
        if hmac.compare_digest(expected, sig_b64):
            return payload
        return None
    return None
    ```

Three problems live here: it **accepts `none`**, it lets the **token's header choose** the
algorithm, and the secret is the guessable word **`s3cr3t`**. You fix all three.

---

## Fix 1 — Reject `none`, and pin the algorithm

Replace the `alg = ...` block above with this. The server now decides the algorithm; the token
does not get a vote, and `none` is simply not `HS256`, so it falls through to rejection:

```python
    # The SERVER decides the algorithm. The token's header does not get to choose.
    # Anything that is not exactly HS256 (including "none") is rejected.
    if header.get("alg") != "HS256":
        return None
    signing_input = f"{h_b64}.{p_b64}".encode()
    expected = _b64url(hmac.new(JWT_SECRET.encode(), signing_input, hashlib.sha256).digest())
    if not hmac.compare_digest(expected, sig_b64):
        return None
    return payload
```

Rebuild and test your **`alg:none`** token:

```bash
docker compose up -d --build
curl -H "Authorization: Bearer <YOUR_ALG_NONE_TOKEN>" http://localhost:8080/api/notes/3
```

!!! success "Checkpoint — the no-signature token is dead"
    It now returns `{"error":"unauthorized"}`. Last class it returned bob's grade. The server no
    longer accepts a token that says "don't check me."

    **But your `s3cr3t`-signed token still works** — it is a valid HS256 signature. That is Fix 2.

---

## Fix 2 — Change the secret, and stop shipping a weak default

The `s3cr3t`-signed forgery still passes because the secret is still `s3cr3t`. Change it — and
remove the weak fallback so the app refuses to run without a real secret.

**1. Generate a long random secret:**

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

**2. Edit `web/app.py`** — no guessable default:

```python
JWT_SECRET = os.environ["JWT_SECRET"]   # required; no weak default
```

**3. Put the secret in the environment**, not the code. Add it to `docker-compose.yml` under the
`web` service (use the value you generated):

```yaml
    environment:
      JWT_SECRET: "PASTE_YOUR_GENERATED_SECRET_HERE"
```

!!! tip "Why not leave it in the code?"
    A secret in the source is in Git, in every clone, in every screen-share. It is not secret. The
    environment keeps it out of the code — and you will see the full version of this idea when the
    course reaches *secrets and environment* (Part 5).

Rebuild and test your **`s3cr3t`-signed** token:

```bash
docker compose up -d --build
curl -H "Authorization: Bearer <YOUR_S3CR3T_SIGNED_TOKEN>" http://localhost:8080/api/notes/3
```

!!! success "Checkpoint — the forged seal no longer matches"
    It now returns `{"error":"unauthorized"}`. Your forged seal was made with the **old** secret.
    The server now signs and checks with a **new** secret, so your seal does not match.

---

## Fix 3 — Confirm you did not break the door

A fix that also blocks real users is not a fix. Log in normally and confirm it still works.

```bash
# log in as alice and read her OWN note through the app
curl -s -c j.jar -d "username=alice&password=alicepw" http://localhost:8080/login >/dev/null
curl -s -b j.jar http://localhost:8080/dashboard | grep -i "grade\|comment" | head
```

Or just open <http://localhost:8080> in the browser and log in as `alice` / `alicepw`. Her
dashboard loads, and the app issues her a **genuine** token signed with the new secret.

!!! success "Checkpoint — real login still works"
    alice logs in and sees her grades. The genuine token the server issues passes its own new
    check. You closed the hole without locking out real users.

---

## What you hand in

```
LOGIN FIX — NotesApp flaw #4 (auth) closed
  Fix 1:  verify_jwt rejects alg:none and pins algorithm to HS256
  Fix 2:  JWT_SECRET is long + random, read from the environment (no s3cr3t, no default)
  Proof:  alg:none token        -> ____ (expect: unauthorized)
          s3cr3t-signed token   -> ____ (expect: unauthorized)
          real alice login      -> ____ (expect: works)
```

Commit your fixed `web/app.py` and the `JWT_SECRET` wiring in `docker-compose.yml`. **Do not commit
the secret's real value if this were production** — for this lab it is fine, but note in your README
that in real life the secret comes from a secrets store, never from a committed file.

---

## Cleanup

```bash
docker compose down     # your data is kept in a volume
```

## Troubleshooting

??? bug "The app won't start after Fix 2: `KeyError: 'JWT_SECRET'`"
    That is the point of removing the default — the app now refuses to run without a real secret.
    Set `JWT_SECRET` in `docker-compose.yml` under the `web` service, then
    `docker compose up -d --build`.

??? bug "A real login now fails too"
    You probably changed the secret but an old browser session still holds a token signed with the
    old one. Log out and log in again — the server issues a fresh token with the new secret.

??? bug "The forged token still works after Fix 1"
    You edited the file but did not rebuild. The image is cached — run `docker compose up -d --build`
    so your new `verify_jwt` is actually in the running container.

---

??? note "Reference fix (open only if you are stuck or at the timer)"
    `web/app.py`, the `verify_jwt` body, after the header/payload are decoded:

    ```python
    if header.get("alg") != "HS256":
        return None
    signing_input = f"{h_b64}.{p_b64}".encode()
    expected = _b64url(hmac.new(JWT_SECRET.encode(), signing_input, hashlib.sha256).digest())
    if not hmac.compare_digest(expected, sig_b64):
        return None
    return payload
    ```

    Near the top:

    ```python
    JWT_SECRET = os.environ["JWT_SECRET"]
    ```

    `docker-compose.yml`, under `web:` → `environment:`:

    ```yaml
      JWT_SECRET: "<a long random value from: python3 -c 'import secrets; print(secrets.token_urlsafe(32))'>"
    ```

    This rejects `alg:none` and every non-HS256 header, verifies the signature against a strong
    secret the token cannot know, and still accepts genuine tokens the server issues.
