# Lab 1 — Audit: is NotesApp's traffic safe in transit?

**Your job:** audit whether NotesApp protects a login as it travels from the browser to the server.
You'll try to read a password off the wire, fix what you find, prove the fix works, and then judge
whether the fix can actually be *trusted*. About 45 minutes, with your pair.

Work inside your `notesapp/` clone. Every command assumes you are in that folder.

!!! abstract "What you hand in — a findings sheet, not a transcript"
    The deliverable is the **Audit summary** at the bottom: your evidence, your severity calls, and
    one trust verdict. The commands below are scaffolding — run them, or have AI drive them. What is
    yours is the evidence you capture and the calls you make on it. That is the part AI can't fake.

!!! warning "Only your own traffic"
    You capture only your own machine's loopback traffic, to your own app. That is the whole target.

!!! tip "Pace check (phases, not answers)"
    | By minute | Phase |
    |---|---|
    | ~15 | Finding 1 captured |
    | ~30 | Remediation applied — HTTPS running |
    | ~40 | Finding 1 re-tested under HTTPS |
    | ~45 | Finding 2 — certificate judged |

---

## Finding 1 — Can someone read a login on the wire?

NotesApp runs on plain HTTP today. You are going to find out, first-hand, what that means for a
password — using **Wireshark**, a free tool anyone can download.

!!! question "Predict first — commit to an answer with your pair"
    You are about to capture your *own* login to NotesApp and look inside the captured traffic.
    **When you open it, will your password be readable?** Write down yes or no, and why.

??? note "How to capture a login (scaffolding — the mechanical steps)"
    **Start the app on plain HTTP:**

    ```bash
    docker compose up -d     # start db + web in the background (-d = detached)
    ```

    Open <http://localhost:8080>, confirm the login page loads — but do **not** log in yet; get
    Wireshark running first.

    **Capture:**

    1. Open **Wireshark**.
    2. Choose the **loopback** interface — macOS `Loopback: lo0` · Linux `lo` · Windows
       `Adapter for loopback traffic capture` (older Npcap calls it `Npcap Loopback Adapter`).
       (NotesApp runs on `localhost`, so loopback is the only interface that
       sees it. Can't see it / nothing captures? See [Troubleshooting](#troubleshooting).)
    3. Filter bar — type this and press Enter (turns green when valid):

        ```
        tcp.port == 8080 and http
        ```

        This tells Wireshark to show **only** traffic on port `8080` (where NotesApp listens) that it
        recognises as **HTTP** — hiding the noise so you can find your login.

    4. Click the blue shark-fin **Start capturing**.
    5. In the browser, log in as `alice` / `alicepw`.
    6. Back in Wireshark, click the red square **Stop capturing**.
    7. Find the row whose `Info` starts with `POST /login HTTP/1.1`. Right-click it → **Follow** →
       **HTTP Stream**. A window opens with the full request as text.

    ??? bug "No `POST /login` row appears"
        Almost always the **wrong interface** — it must be loopback (`lo0` / `lo` / Windows's
        `Adapter for loopback traffic capture`). Stop, reselect it, start again, log in again. Still
        nothing (common on Windows +
        Docker Desktop)? Use the sample capture in [Troubleshooting](#troubleshooting).

!!! success "📏 Record — what did the capture show?"
    Look through the Follow-Stream window for the login you just sent.

    - Your username and password — **readable, yes or no?**
    - If yes, copy the exact line you found: ___________________________
    - Take a screenshot. It is your "before" evidence.

??? question "Done looking? Open this to debrief — not before"
    Was your prediction right? And the real question: **who, in the real world, could have read what
    you just read?**

    ??? success "The answer"
        The username and password are right there in clear text — no decoding, no tool beyond a free
        packet sniffer. On shared WiFi (an airport, a café), **anyone else on that network running the
        same free tool reads your login**, with no special access and in under two minutes. Plain HTTP
        does not hide anything; it just assumes nobody is looking.

**Write Finding 1** (one line each — goes in your Audit summary):

```
FINDING 1 — login in transit (plain HTTP)
  What's exposed:  ________________________________
  Evidence:        (your before-screenshot: the ______ line in the HTTP stream)
  Who can read it: ________________________________
  Severity:        (Low / Med / High) because _______
  Fix (1 line):    ________________________________
```

---

## Remediation — put HTTPS in front of NotesApp

You add an **nginx** reverse proxy in front of NotesApp. It terminates TLS (does the HTTPS
encryption) for the browser, and talks plain HTTP to the Flask app only **inside** the Docker
network — the leg an outside attacker can never reach.

??? note "How to turn on HTTPS (scaffolding)"
    **1. Generate a self-signed certificate** (we sign it ourselves instead of paying a Certificate
    Authority — hold that thought for Finding 2):

    ```bash
    mkdir -p nginx/certs       # folder nginx will read the cert + key from
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout nginx/certs/notesapp.key \
      -out nginx/certs/notesapp.crt \
      -subj "/CN=localhost"
    ```

    What each part of the `openssl` command means:

    - `req -x509` — make a **self-signed certificate** (not a request sent to a real CA).
    - `-nodes` — "no DES": don't passphrase-encrypt the private key, so nginx can read it unattended.
    - `-days 365` — valid for a year.
    - `-newkey rsa:2048` — generate a fresh 2048-bit RSA key pair at the same time.
    - `-keyout` / `-out` — where to write the **private key** and the **certificate**.
    - `-subj "/CN=localhost"` — the identity the cert claims: the domain `localhost`. (`CN` = Common
      Name. This is the field your browser checks against the address bar — remember it for Finding 2.)

    `notesapp.key` is the **private key** (never share it, never commit it); `notesapp.crt` is the
    **certificate** (the public key + the domain it claims).

    **2. Write `nginx/nginx.conf`** with exactly this:

    ```nginx
    # --- Plain HTTP (port 80): send everyone to HTTPS instead ---
    server {
        listen 80;                                   # listen for plain http
        server_name localhost;
        return 301 https://$host$request_uri;        # permanent redirect to the https:// version
    }

    # --- HTTPS (port 443): the encrypted entrance ---
    server {
        listen 443 ssl;                              # listen for https, enable TLS
        server_name localhost;

        ssl_certificate     /etc/nginx/certs/notesapp.crt;   # the cert you just generated
        ssl_certificate_key /etc/nginx/certs/notesapp.key;   # its private key

        location / {                                 # for every request path...
            proxy_pass http://web:5000;              # ...forward to the Flask app, over the PRIVATE
                                                     #    docker network (plain http, but unreachable
                                                     #    from outside the containers)
            proxy_set_header Host $host;             # pass the original hostname through
            proxy_set_header X-Real-IP $remote_addr; # pass the client's IP through
        }
    }
    ```

    In one sentence: port 80 bounces everyone to HTTPS; port 443 does the TLS encryption with your
    cert; `proxy_pass` hands the decrypted request to Flask **inside Docker**, the one leg an attacker
    on the outside can never see.

    **3. Create `docker-compose.https.yml`** with this content. It adds nginx in front, and stops
    publishing the app's own port so the only way in is through HTTPS:

    ```yaml
    services:
      web:
        # Compose MERGES ports across files, so clear the base 8080 explicitly -
        # otherwise the app stays reachable over plain HTTP and the fix is pointless.
        ports: !reset []
        expose:
          - "5000"

      nginx:
        image: nginx:1.27
        depends_on:
          - web
        ports:
          - "443:443"
          - "80:80"
        volumes:
          - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
          - ./nginx/certs:/etc/nginx/certs:ro
    ```

    Then launch:

    ```bash
    docker compose down          # stop the plain-HTTP stack first (frees the ports)
    # bring it up with BOTH files: the base stack + the https overlay on top.
    # -d = background (detached), --build = rebuild images if anything changed.
    docker compose -f docker-compose.yml -f docker-compose.https.yml up -d --build
    ```

    `docker compose ps` should now show three containers: `db`, `web`, `nginx` — and `web` no longer
    lists a published `8080` port.

    **4. Open <https://localhost>.** Your browser shows a warning and makes you click **Advanced →
    Proceed**. Do that for now — *why* it warns is exactly Finding 2, so don't dismiss the question,
    just get past the screen.

!!! success "Checkpoint — HTTPS is running"
    The NotesApp login page loads over `https://localhost` (after clicking through the warning).

---

## Finding 1, re-tested — did the fix actually close it?

A fix you cannot verify is not a fix. Prove it with the same attack.

!!! question "Predict first"
    You repeat the **exact** same capture — same login, same tool — but now the connection is HTTPS.
    **Will your password be readable this time?** Commit to an answer.

??? note "How to re-capture under HTTPS (scaffolding)"
    1. In Wireshark, set the filter to `tcp.port == 443` (443 = the HTTPS port; no `http` keyword
       this time, because the traffic is now TLS and Wireshark can't read it as HTTP) and start
       capturing.
    2. Log in again at <https://localhost> as `alice` / `alicepw`.
    3. Stop. Right-click a row → **Follow** → **TLS Stream** (there is no HTTP stream to follow now).

!!! success "📏 Record — what did this capture show?"
    - Can you find your username or password anywhere in the stream — **yes or no?**
    - Screenshot it. This is your "after" evidence, next to the "before" from Finding 1.

??? success "Done looking? Open to confirm what you should have seen"
    The stream is unreadable bytes — no `username=`, no `password=`, nowhere. The data is still in
    there, but encrypted; Wireshark has no session key, and neither does an attacker on the wire.
    Same password, same tool, same two minutes — and now it leaks nothing. **That** is what HTTPS
    bought. Add both screenshots to your audit as before/after evidence.

---

## Finding 2 — can you *trust* this HTTPS? (judge)

Your traffic is encrypted — you just proved it. But your browser still threw a scary warning to get
here. Those two facts sitting together are the whole lesson.

!!! question "Think it through with your pair before you investigate"
    The connection is encrypted, yet the browser warned you anyway. So:

    - Is "the connection is **encrypted**" the same thing as "the server is **who it claims to be**"?
    - What is the warning actually complaining about, if not the encryption?

??? note "How to investigate (scaffolding) — compare two certificates"
    1. Visit a real site, e.g. <https://github.com>. Click the padlock → view the certificate. Note
       the **Issued by** field and whether there's a chain of trust.
    2. Now view your own certificate at <https://localhost> the same way. Note **its** issuer.

!!! success "📏 Record — the one difference that matters"
    - GitHub's certificate — **Issued by:** ______________________
    - Your certificate — **Issued by:** ______________________

??? question "Open to check your judgment — after you've compared them"
    What single piece of trust does yours lack that GitHub's has — and why does encryption alone not
    provide it?

    ??? success "The answer"
        GitHub's cert is **signed by a Certificate Authority your browser already trusts**, which
        verified that GitHub really controls `github.com`. Yours is **self-signed** — it vouches for
        itself, which anyone (including an attacker) can do in ten seconds. Encryption scrambles the
        data, but it says nothing about *who is on the other end*. The browser warning is not about
        weak encryption; it is about **unproven identity**. Encrypted ≠ trusted. A real deployment
        needs a CA-signed certificate — that is the passport-for-a-public-key from Class 4.

**Write Finding 2 + the verdict** (goes in your Audit summary):

```
FINDING 2 — certificate trust
  GitHub issued by:  ____        Yours issued by:  ____
  The trust yours lacks:  ______________________________
  Verdict — is "encrypted" the same as "safe to trust"? Why (one sentence):
  ____________________________________________________________
```

---

## Audit summary — hand this in

```
NOTESAPP TRANSIT AUDIT — pair: ____________

FINDING 1 — login over plain HTTP
  Exposed:      ____ (before-screenshot: the ____ line)
  Who reads it: ____
  Severity:     ____ because ____
  Fix:          ____

FIX VERIFIED — re-captured under HTTPS: password readable?  ____ (after-screenshot)

FINDING 2 — certificate trust
  GitHub issued by: ____   Yours issued by: ____
  Trust yours lacks: ____
  Verdict (encrypted vs trusted): ____

STATE: NotesApp now served over HTTPS (self-signed, for the lab).  ✅
```

??? tip "Finished early? (optional) — two more things to break"
    **Wrong domain name.** Edit `nginx/nginx.conf` so `server_name` is `notesapp.local`, regenerate
    the cert with `-subj "/CN=notesapp.local"`, and revisit `https://localhost`. Read the *new*
    warning — it is a **name mismatch**, a different failure than "untrusted issuer". Phishing sites
    trigger exactly this by reusing a cert for the wrong domain.

    **Downgrade check.** With HTTPS running, try `curl -v http://localhost` (no `s`). Confirm you are
    redirected (a `301` and a `Location: https://...` header) instead of quietly served over plain
    HTTP.

## Cleanup (end of lab)

Back to plain HTTP for whatever comes next:

```bash
# stop the https stack (both files, so nginx is removed too)...
docker compose -f docker-compose.yml -f docker-compose.https.yml down
docker compose up -d     # ...and start the plain-HTTP stack again
```

Your database is untouched — it lives in the `pgdata` volume, which neither command deletes.

## Troubleshooting

??? bug "'bind: address already in use' on port 443 or 80"
    Something else is using that port (another local server, or the plain stack still up). Run
    `docker compose down` fully before the HTTPS overlay `up`.

??? bug "Wireshark shows nothing on loopback (Windows especially)"
    **Common cause — Npcap without loopback support (Windows).** Re-run the Npcap installer from
    [npcap.com](https://npcap.com/#download) alone, tick **"Support loopback traffic capture"**,
    reboot, reopen Wireshark — an interface called **`Adapter for loopback traffic capture`** appears
    (older Npcap named it `Npcap Loopback Adapter`). Capture on it.

    **Still nothing** (Docker Desktop's WSL2 VM can hide loopback traffic from Npcap)? Use the sample
    capture — a real HTTP login, so you can complete Finding 1 without capturing live:

    **⬇ Download: [`login-http.pcap`](assets/login-http.pcap)** — open in Wireshark (File → Open),
    right-click the `POST /login` row → **Follow** → **HTTP Stream**, and read the request body.

    **Cross-OS cross-check:** browser DevTools (F12) → **Network** → log in → click the `login`
    request → **Payload**. The credentials show there too. (Caveat: DevTools shows the browser's own
    copy, so it looks the same over HTTPS — it proves the request is plaintext, but only Wireshark
    proves what is readable *on the wire*. That gap is exactly Finding 1 vs its re-test.)

??? bug "'docker compose' complains about nginx/certs not found"
    You skipped the certificate step, or are in the wrong directory. Every command must run inside
    `notesapp/`.

??? bug "Error mentioning '!reset' or 'reset' when starting the HTTPS overlay"
    Your Docker Compose is older than 2.24 (when `!reset` was added). Update Docker Desktop (the
    easiest fix), then retry. Check your version with `docker compose version`.