# Lab — Your words run in someone else's browser, then take the power away

**Your job, in one hour:** leave a grade comment that is **not a comment** — it is a little program.
Then watch it **run by itself in another student's browser**, with that student's login. Then **shut
it off** by putting back one line of safety the app had switched off. About 50 minutes, with your
pair.

Last class was injection on the way **in**: what you type becomes a command on the server (SQL, the
shell). Today is injection on the way **out**: what the server **stores** becomes **code in a
victim's web page**. One flaw: **stored cross-site scripting (XSS), NotesApp flaw #10.**

!!! abstract "What you hand in — one trap, then the fix"
    1. Proof your `<script>` comment **ran** in a student's dashboard (stored XSS).
    2. Your fixed templates, where the exact same comment is now **shown as text, not run**.
    3. Proof a **normal** comment still displays correctly after the fix.

!!! warning "Only your own app"
    You attack and fix **only** your own NotesApp clone, on your own machine. The accounts and grades
    here are seed data, not real people.

---

## The one idea for the whole hour

A browser does not just **show** a page — it **obeys** it. If a page contains `<script>…</script>`,
the browser **runs** it. NotesApp shows each grade comment like this, in `dashboard.html`:

```html
{% autoescape false %}{{ n.comment }}{% endautoescape %}
```

That `autoescape false` is the whole bug. Normally the template library is **safe by default**: it
turns `<` into `&lt;`, so a script tag is **displayed as text**, never run. This line switches that
safety **off** for the comment — so whatever a teacher stores is handed to every reader's browser as
**live code**.

Think of it as a **glass case**: by default your comment sits behind glass — readers see every
character, including `<script>`, but nothing reaches out. `autoescape false` **removes the glass**.
The fix is to **put the glass back**.

And it is **stored**: the teacher writes it **once**; it fires for **every student** who later opens
that page, with the attacker **not even present**.

---

## Part 1 — Plant a script in a comment and watch it fire (≈ 8 min)

Start the app.

```bash
docker compose up -d
```

Log in as a **teacher** (teachers can write grade comments) and keep the cookie:

```bash
curl -s -c teacher.jar -d "username=prof.turing&password=teach123" \
     http://localhost:8080/login -o /dev/null -w "teacher login: HTTP %{http_code}\n"
```

Now save a grade comment on **note 1** — but instead of a comment, send a script:

```bash
curl -s -b teacher.jar \
     --data-urlencode "note_id=1" \
     --data-urlencode "grade=16.2" \
     --data-urlencode "comment=<script>alert('XSS by Eve')</script>" \
     http://localhost:8080/grades -o /dev/null -w "comment saved: HTTP %{http_code}\n"
```

!!! question "Predict first — commit with your pair"
    Note 1 belongs to the student **alice**. In a moment alice opens her own dashboard. What happens
    in **her** browser — is the script **shown as text**, does it cause an **error**, or does it
    **run**? Write your guess before you look.

See what the server sends to alice. Log in as alice and fetch her dashboard:

```bash
curl -s -c alice.jar -d "username=alice&password=alicepw" \
     http://localhost:8080/login -o /dev/null
curl -s -b alice.jar http://localhost:8080/dashboard | grep -o "<script>.*</script>"
```

!!! danger "Checkpoint — the comment comes back as live code"
    `grep` prints your **literal** `<script>alert('XSS by Eve')</script>`, unescaped, sitting inside
    alice's page. A real browser receiving this does not show it — it **runs** it. To see it fire for
    real, open <http://localhost:8080> in your browser, log in as **alice / alicepw**, and open the
    dashboard: the alert pops. This is **stored XSS**, flaw #10. The teacher typed it once; it now
    runs for whoever opens that note.

??? tip "Why this is dangerous (optional) — it runs AS alice"
    The script runs with **alice's** session, not yours. Swap the harmless `alert(...)` for code that
    reads `document.cookie`, or silently calls `/grades` to change a grade, and it acts **as alice**,
    from alice's browser, while she just looks at her own page. One stored comment borrows every
    reader's account.

---

## Part 2 — Put the glass back: the fix (≈ 8 min)

The bug is one flag, in **two** templates. Open `web/templates/dashboard.html` (around line 15) and
`web/templates/teacher.html` (around line 47). Both have:

```html
<td>{% autoescape false %}{{ n.comment }}{% endautoescape %}</td>
```

The fix is **not** "ban the word script" — you can never list every dangerous payload. The fix is to
**stop turning off the safety**. Delete the `autoescape false` wrapper and let the template escape by
default:

```html
<td>{{ n.comment }}</td>
```

Do this in **both** files. Rebuild and fetch alice's dashboard again:

```bash
docker compose up -d --build
curl -s -b alice.jar http://localhost:8080/dashboard | grep -o "&lt;script&gt;.*&lt;/script&gt;" \
  && echo "  -> escaped: shown as text, NOT run"
```

!!! success "Checkpoint — the script is now inert text"
    The page now contains `&lt;script&gt;alert('XSS by Eve')&lt;/script&gt;` — the browser **displays**
    the characters and **never runs** them. Reload alice's dashboard in the browser: no alert, just
    the literal text shown in the comment cell. The glass is back.

---

## Part 3 — Prove it: shown, not run (≈ 6 min)

Confirm the exact attack from Part 1 is now dead. Re-save the same script comment as the teacher, then
view as alice:

```bash
curl -s -b teacher.jar \
     --data-urlencode "note_id=1" --data-urlencode "grade=16.2" \
     --data-urlencode "comment=<script>alert('XSS by Eve')</script>" \
     http://localhost:8080/grades -o /dev/null

# the raw <script> must NOT appear; the escaped &lt;script&gt; must:
curl -s -b alice.jar http://localhost:8080/dashboard | grep -o "<script>alert" \
  && echo "STILL VULNERABLE" || echo "safe: no live <script> in the page"
```

!!! success "Checkpoint — same input, no code"
    `grep` finds no live `<script>alert` — only the escaped version is in the page. The browser shows
    the comment and runs nothing. Open it in the browser to confirm: the alert never fires.

---

## Part 4 — Prove real comments still display right (≈ 4 min)

A fix that mangles normal comments is not a fix. Teachers must still be able to write `<`, `>` and
`&` in plain text (e.g. "score < 10, see me & retry").

```bash
curl -s -b teacher.jar \
     --data-urlencode "note_id=1" --data-urlencode "grade=11.0" \
     --data-urlencode "comment=score < 10 on part 1 & 2, see me" \
     http://localhost:8080/grades -o /dev/null
curl -s -b alice.jar http://localhost:8080/dashboard | grep -o "score &lt; 10.*retry\|score &lt; 10[^<]*"
```

!!! success "Checkpoint — normal text is fine"
    The comment displays as the teacher wrote it. The `<` shows as a less-than sign on the page (sent
    as `&lt;`), not as the start of a tag. Normal comments work; only code is neutralised.

---

## What you hand in

```
STORED XSS FIX — NotesApp flaw #10 closed

  Break:  comment on note 1 =  <script>alert('XSS by Eve')</script>
          opened as alice -> ____ (expect: alert runs / raw <script> in page)

  Fix:    removed {% autoescape false %} in dashboard.html AND teacher.html

  Closed: same comment now shows as text -> ____ (expect: &lt;script&gt;, no alert)

  Real use still works:  "score < 10 ... & retry" displays correctly [ ]
```

Commit your fixed `web/templates/dashboard.html` and `web/templates/teacher.html`.

!!! tip "The deeper lesson — escape on the way out"
    Last class: never build a command by gluing user input (SQL, the shell) — that is injection on the
    way **in**. This class: never hand user text to the browser as **code** — escape it — that is
    injection on the way **out**. Same family, both directions: **data stays data.** If you genuinely
    need to allow some HTML (bold, links), do **not** disable escaping — sanitise with an allow-list
    (e.g. the `bleach` library) and keep only safe tags.

---

## Cleanup

```bash
docker compose down     # your data is kept in a volume
```

## Troubleshooting

??? bug "The alert still fires after the fix"
    You edited one template but not both, or did not rebuild. `dashboard.html` **and** `teacher.html`
    both have the `autoescape false` line. Run `docker compose up -d --build` and re-check with the
    `grep` in Part 3 — the live `<script>alert` must be gone.

??? bug "`grep` finds nothing in Part 1"
    Your template may wrap the cell differently. Drop the `| grep ...` and read the raw HTML from
    `curl -s -b alice.jar http://localhost:8080/dashboard` — look at the comment cell and check
    whether it holds a real `<script>` tag (vulnerable) or `&lt;script&gt;` (safe).

??? bug "Comments look empty after the fix"
    You removed too much — keep `{{ n.comment }}`. Only the `{% autoescape false %}` and
    `{% endautoescape %}` wrapper should go.

??? bug "The alert never fired even before the fix"
    `curl` never runs JavaScript — it only shows you the HTML. To *see* it execute, open the page in a
    real browser logged in as alice. The `grep` check (raw `<script>` present) is the reliable
    command-line proof.

---

??? note "Reference fix (open only if you are stuck or at the timer)"
    `web/templates/dashboard.html` (line ~15) and `web/templates/teacher.html` (line ~47):

    ```html
    <!-- BEFORE (flaw #10): escaping switched off, comment handed over as live code -->
    <td>{% autoescape false %}{{ n.comment }}{% endautoescape %}</td>

    <!-- AFTER: default autoescape restored — <script> is shown as text, never run -->
    <td>{{ n.comment }}</td>
    ```

    One idea: let the template escape by default. The browser then **shows** every character of the
    comment and **runs** none of it. If rich text is truly needed, sanitise server-side with an
    allow-list (`bleach`) instead of disabling escaping.
