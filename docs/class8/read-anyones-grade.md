# Lab — Read anyone's grade, then lock the door

**Your job, two halves in one hour:** first **read a classmate's grade** you were never meant to
see — by changing one number in a URL. Then **close the hole** so a note that is not yours is
refused. About 50 minutes, with your pair.

Last class you made the token **trustworthy** — the server finally knows *who* you are. Today you
find out that knowing who you are is not enough: the app never checks *what you are allowed to
see*. Those are two different questions.

!!! abstract "What you hand in — the proof you got in, then the fixed handler"
    1. The grade of a student who is **not you**, read through `/api/notes/<id>`.
    2. Your fixed `api_notes` function, where that same request now returns **not found**, while
       your own note and the teacher view still work.

!!! warning "Only your own app"
    You attack and fix **only** your own NotesApp clone, on your own machine. The ids and grades
    here are seed data, not real people.

---

## Part 1 — Walk into the wrong locker (≈ 20 min)

Start the app and log in as a **student**.

```bash
docker compose up -d
```

Open <http://localhost:8080> and log in as `alice` / `alicepw`.

Her dashboard shows **her own** notes. Each note has an id. Alice owns ids **1** and **2**. The
page fetches them from a small API:

```text
GET /api/notes/1     ← alice's own note
```

!!! question "Predict first — commit with your pair"
    The URL ends in a number you can change. **What happens if you ask for id 3** — a note that is
    not yours? Write your guess (error? empty? someone else's grade?) before you run it.

Now ask for a note that is **not** alice's. Grab her session cookie and walk the ids:

```bash
# log in as alice, keep her cookie
curl -s -c alice.jar -d "username=alice&password=alicepw" http://localhost:8080/login >/dev/null

# her own note — allowed, expected
curl -s -b alice.jar http://localhost:8080/api/notes/1

# bob's note — she was NEVER meant to see this
curl -s -b alice.jar http://localhost:8080/api/notes/3
```

!!! danger "Checkpoint — you just read someone else's grade"
    Id 3 returns **bob's** grade and comment, with his name on it — while you are logged in as
    alice. You are fully authenticated. The server checked your wristband at the door. It just
    never checked **whose locker you opened**.

    Keep going: `4`, `5`, `6`… each number is another student's grade. One `for` loop reads the
    **whole class**:

    ```bash
    for id in $(seq 1 10); do
      echo -n "note $id: "; curl -s -b alice.jar http://localhost:8080/api/notes/$id
      echo
    done
    ```

This is **IDOR** — Insecure Direct Object Reference, NotesApp flaw #7. The object (a note) is named
directly by a number in the URL, and the server hands it over **without checking it is yours**.

---

## Part 2 — Find the hole in the code (≈ 10 min)

Open `web/app.py` and find the handler:

```python
@app.route("/api/notes/<int:note_id>")
def api_notes(note_id):
    ident = current_identity()
    if not ident:
        return jsonify({"error": "unauthorized"}), 401
    # ... fetch the note by id ...
    note = cur.fetchone()
    if not note:
        return jsonify({"error": "not found"}), 404
    note["grade"] = float(note["grade"]) if note["grade"] is not None else None
    return jsonify(note)
```

Read it closely. It asks **one** security question — "are you logged in?" (`if not ident`). It
never asks the **second** question — "is this note yours?"

!!! note "Two questions, not one"
    - **Authentication** — *who are you?* Answered by the token/session. You fixed this last class.
    - **Authorization** — *what are you allowed to see?* **Never asked here.** That is the bug.

    A door with a working lock is useless if, once inside, every room is unlocked.

The note row already carries everything you need: it has a `student_id` — the owner. And `ident`
already tells you **your** `user_id`. The fix just compares the two.

---

## Part 3 — Lock the locker (≈ 15 min)

Add the ownership check, right after the note is fetched. A note that is not yours is refused —
**unless** you are a teacher or admin, who are allowed to see every note (that is their job).

```python
    note = cur.fetchone()
    if not note:
        return jsonify({"error": "not found"}), 404

    # Flaw #7 fix: you may only read your OWN note.
    # Teachers and admins are allowed to read any note; a student is not.
    if note["student_id"] != ident["user_id"] and ident.get("role") not in ("teacher", "admin"):
        return jsonify({"error": "not found"}), 404
```

Rebuild and test the **exact** request that leaked bob's grade:

```bash
docker compose up -d --build
curl -s -b alice.jar http://localhost:8080/api/notes/3
```

!!! success "Checkpoint — the wrong locker is now closed"
    Id 3 now returns `{"error":"not found"}`. Alice is still logged in, still authenticated — but
    the note is not hers, so she does not get it. The second question finally gets asked.

!!! question "Why `not found` (404) and not `forbidden` (403)?"
    Both close the hole. But **403 "forbidden"** tells the attacker *"this id exists, you're just
    not allowed."* **404 "not found"** tells them nothing — they cannot even map which ids are real.
    When in doubt, do not confirm the object exists. (403 is accepted for this lab; say which you
    chose in your hand-in.)

---

## Part 4 — Confirm you did not lock out real users (≈ 5 min)

A fix that also blocks the legitimate owner and the teacher is not a fix.

```bash
# alice reads her OWN note — must still work
curl -s -b alice.jar http://localhost:8080/api/notes/1

# a teacher reads any note — must still work
curl -s -c prof.jar -d "username=prof.turing&password=teach123" http://localhost:8080/login >/dev/null
curl -s -b prof.jar http://localhost:8080/api/notes/3
```

!!! success "Checkpoint — owners and teachers unaffected"
    Alice still reads note 1 (hers). prof.turing still reads note 3 (a teacher may). Only the
    **cross-student** read — alice reaching for bob's note — is now refused.

---

## What you hand in

```
IDOR FIX — NotesApp flaw #7 (broken access control) closed
  Proof you got in:   as alice, /api/notes/3 returned bob's grade: ____ (grade + comment)
  Fix:                api_notes refuses a note whose student_id is not the caller's,
                      unless the caller is a teacher/admin
  Code chosen:        returns  [ ] 404 not found   [ ] 403 forbidden
  Proof it's closed:  as alice, /api/notes/3   -> ____ (expect: not found)
                      as alice, /api/notes/1   -> ____ (expect: her own note)
                      as prof.turing, /api/notes/3 -> ____ (expect: works)
```

Commit your fixed `web/app.py`.

!!! tip "The deeper lesson — check ownership on the server, for every object"
    `/api/notes/<id>` was one door. The same bug hides anywhere an id in a URL names something that
    belongs to one user: `/invoice/<id>`, `/message/<id>`, `/order/<id>`. The rule is the same
    every time: **the server decides what you may see, from your identity — never trust that the
    URL you were given is the only one you will ask for.**

---

## Cleanup

```bash
docker compose down     # your data is kept in a volume
```

## Troubleshooting

??? bug "`/api/notes/3` still returns bob's grade after the fix"
    You edited the file but did not rebuild. Run `docker compose up -d --build` so your new
    `api_notes` is actually in the running container.

??? bug "Now `/api/notes/1` is refused for alice too"
    Your comparison is backwards or using the wrong field. `ident["user_id"]` is the caller;
    `note["student_id"]` is the owner. They are **equal** for alice's own note — that case must pass.

??? bug "The teacher is now blocked"
    Your role check is missing or misspelled. `ident.get("role")` must be allowed to be
    `"teacher"` or `"admin"`. Confirm the token/session actually carries the role.

---

??? note "Reference fix (open only if you are stuck or at the timer)"
    `web/app.py`, in `api_notes`, immediately after the `if not note:` block:

    ```python
    if note["student_id"] != ident["user_id"] and ident.get("role") not in ("teacher", "admin"):
        return jsonify({"error": "not found"}), 404
    ```

    This asks the second question — *is this note yours?* — that the handler was missing. The note
    already carries its owner (`student_id`); the caller's identity already carries `user_id` and
    `role`. A student gets only their own notes; a teacher or admin still gets any note.
