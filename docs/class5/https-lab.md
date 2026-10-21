# Lab 1 — HTTPS for NotesApp

**Goal:** watch a real password travel in clear text over HTTP. Then turn on HTTPS for NotesApp
yourselves, and watch the same password become unreadable. **About 45 minutes.**

Work inside your `notesapp/` clone. Every command below assumes you are in that folder.

!!! tip "Pace check — are you on track?"
    Use this to check yourself, not to rush.

    | By minute | You should be at |
    |---|---|
    | ~15 | Part 1 done — password found in the HTTP capture |
    | ~30 | Part 2 done — certificate generated, `nginx.conf` written, HTTPS running |
    | ~40 | Part 3 done — second capture shows no readable password |
    | ~45 | Part 4 done — certificates compared |

    Behind at the 30-minute mark? Skip straight to Part 3 by copying the ready-made `nginx.conf`
    from [Part 2, Step 2](#step-2-write-the-nginx-configuration) exactly as written, and come back to
    reading it line by line later — seeing the result matters more today than typing every line
    yourselves.

## Part 0 — Start the app the old way (plain HTTP)

```bash
docker compose up -d
```

Open <http://localhost:8080> in your browser. Confirm the app loads and you can see the login page.
Do **not** log in yet — we want Wireshark running first.

!!! success "Checkpoint"
    The login page is visible at `http://localhost:8080` (note: `http`, not `https`).

## Part 1 — Capture the password over HTTP

1. Open **Wireshark**.
2. In the interface list, choose the **loopback** interface:
    - macOS: `Loopback: lo0`
    - Linux: `lo`
    - Windows: `Npcap Loopback Adapter`. If you do not see it, Npcap was installed without loopback
      support — see [the Windows fix in Troubleshooting](#troubleshooting). On Windows, `localhost`
      traffic can also be invisible because NotesApp runs inside Docker Desktop's VM — if you get
      stuck, use the **sample capture fallback** in Troubleshooting so you can still do the exercise.
3. In the filter bar at the top, type this and press Enter (it turns green when valid):

    ```
    tcp.port == 8080 and http
    ```

4. Click the blue shark-fin **Start capturing** button.
5. Switch to your browser. Log in to NotesApp as:

    ```
    username: alice
    password: alicepw
    ```

6. Switch back to Wireshark. Click the red square **Stop capturing** button.

!!! success "Checkpoint"
    You should see several rows. Look for one with `Info` starting with `POST /login HTTP/1.1`.

    ??? bug "I see nothing / no POST row — now what?"
        Almost always the **wrong interface**. The capture only works on **loopback** (`lo0` / `lo` /
        `Npcap Loopback Adapter`), because NotesApp runs on `localhost`. Fixes, in order:

        1. Stop the capture, go back to step 2, pick the loopback interface, start again, log in again.
        2. Check the filter reads exactly `tcp.port == 8080 and http` and is green. If your app runs on
           a different port, match it.
        3. Still nothing after logging in (common on Windows + Docker Desktop)? Use the **sample
           capture** in [Troubleshooting](#troubleshooting) — real captured traffic you can open and
           complete the exercise with.

7. Right-click that row → **Follow** → **HTTP Stream**.
8. A new window opens showing the full request and response as text. Scroll until you find a line
   that looks like:

    ```
    username=alice&password=alicepw
    ```

!!! danger "This is the password, in clear text"
    Readable by anyone on the network. Take a screenshot — you will need it for your Project 3 notes
    if your audit app has the same flaw.

!!! question "Discuss with your neighbor"
    If this capture happened on public WiFi at an airport, who could read this?

    ??? success "Answer"
        Anyone else connected to the same network who runs a packet capture tool, exactly like you
        just did. No special skill or secret tool required — Wireshark is free and this capture took
        under two minutes.

## Part 2 — Build the HTTPS version

We add an **nginx** reverse proxy in front of NotesApp that terminates TLS (handles the HTTPS
encryption) and forwards plain traffic to the app over the private Docker network only.

### Step 1 — Generate a self-signed certificate

In a real deployment, your certificate is signed by a trusted Certificate Authority (Class 4). For
this lab, we generate our own — called **self-signed**. Your browser will warn you about it; that
warning is expected and is the whole point of the check at the end of this lab.

```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nginx/certs/notesapp.key \
  -out nginx/certs/notesapp.crt \
  -subj "/CN=localhost"
```

This creates two files:

- `nginx/certs/notesapp.key` — the **private key**. Never share this, never commit it to Git.
- `nginx/certs/notesapp.crt` — the **certificate**, containing the public key and the domain name
  it claims (`CN=localhost`).

!!! success "Checkpoint"
    `ls nginx/certs` shows both files.

### Step 2 — Write the nginx configuration

Create `nginx/nginx.conf` with this exact content:

```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/certs/notesapp.crt;
    ssl_certificate_key /etc/nginx/certs/notesapp.key;

    location / {
        proxy_pass http://web:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

What this does, line by line:

- First `server` block: any plain HTTP request on port 80 gets redirected to HTTPS.
- Second `server` block: HTTPS on port 443, using the certificate and key from Step 1.
- `proxy_pass http://web:5000` — nginx talks to the Flask app in plain HTTP, but only **inside** the
  Docker network, which the attacker cannot reach from outside. Only the nginx↔browser leg is
  encrypted, and that is the leg that matters: the one crossing the public internet.

### Step 3 — Launch with the HTTPS overlay

The repo already ships a stub: `docker-compose.https.yml.example`. Copy it (do not edit the
`.example` file directly, so the original stays a clean reference):

```bash
cp docker-compose.https.yml.example docker-compose.https.yml
docker compose down
docker compose -f docker-compose.yml -f docker-compose.https.yml up -d --build
```

!!! success "Checkpoint"
    `docker compose ps` shows three containers: `db`, `web`, and now `nginx`.

### Step 4 — Open the HTTPS site

Go to <https://localhost> (port 443, the default HTTPS port — no `:443` needed in the URL).

Your browser will show a warning: **"Your connection is not private"** (Chrome) or similar. This is
expected — your browser does not trust our self-signed certificate, because no recognized
Certificate Authority vouched for it (exactly Class 4).

Click **Advanced** → **Proceed to localhost (unsafe)** (wording differs slightly per browser).

!!! success "Checkpoint"
    The NotesApp login page loads, now with a padlock icon (crossed-out or "Not Secure" label — that
    label is about **trust**, not encryption; the connection is still encrypted, it is just
    encrypted to an identity no one vouched for).

## Part 3 — Capture again: this time it should fail

1. In Wireshark, change the filter to:

    ```
    tcp.port == 443
    ```

2. Start capturing.
3. Log in again at <https://localhost> with the same `alice` / `alicepw`.
4. Stop capturing.
5. Try the same trick as before: right-click a row → **Follow** → **TLS Stream** (not HTTP Stream —
   there is no HTTP stream to follow anymore, the whole thing is TLS).

**What you should see:** unreadable binary noise. No `username=`, no `password=`, nowhere. Wireshark
cannot decrypt this traffic — it does not have the session secret, and neither does an attacker on
the network.

!!! success "Checkpoint"
    You cannot find the password anywhere in the capture. That is success — screenshot this as your
    "after" evidence, next to the "before" screenshot from Part 1.

## Part 4 — Compare a self-signed certificate to a real one

1. In your browser, go to any real HTTPS site you trust, e.g. <https://github.com>.
2. Click the padlock icon → view certificate details (menu wording differs per browser — look for
   "Connection is secure" → "Certificate is valid").
3. Note what you see: issued by a named Certificate Authority (e.g. "DigiCert", "Let's Encrypt"), a
   full chain of trust, valid dates, and the exact domain name `github.com`.
4. Now view the certificate for <https://localhost> (the one you just made). Note: issuer is
   `localhost` itself (self-signed — it vouches for itself, which is worth nothing), no chain of
   trust, and your browser flagged it.

!!! question "Write one sentence in your lab notes"
    What is the one piece of trust that is missing from our self-signed certificate, that GitHub's
    certificate has?

    ??? success "Answer"
        A signature from a Certificate Authority that your browser already trusts, confirming the
        certificate owner really controls that domain.

## Bonus — finished early? (optional)

??? tip "Wrong domain name — a different browser warning"
    Edit `nginx/nginx.conf` so `server_name` is `notesapp.local` instead of `localhost`, regenerate
    the certificate with `-subj "/CN=notesapp.local"`, then visit `https://localhost` again (not
    `notesapp.local`). Read the new warning carefully — this one is a **name mismatch**, a different
    failure than "untrusted issuer". Real phishing sites sometimes trigger exactly this warning by
    reusing a certificate for the wrong domain — now you know what it means.

??? tip "Downgrade check"
    With the HTTPS overlay still running, try `curl -v http://localhost` (port 80, no `s`). Confirm
    you get redirected to HTTPS automatically (look for `301` and a `Location: https://...` header)
    rather than quietly served over plain HTTP.

## Cleanup (end of lab)

If the next lab expects plain HTTP on port 8080, switch back:

```bash
docker compose -f docker-compose.yml -f docker-compose.https.yml down
docker compose up -d
```

Your database data is unaffected either way — it lives in the `pgdata` volume, which neither command
deletes.

## Troubleshooting

??? bug "'bind: address already in use' on port 443 or 80"
    Something else on your machine is using that port (another local web server, or Docker already
    running the plain stack). Run `docker compose down` first, fully, before the HTTPS overlay `up`.

??? bug "Wireshark shows nothing on loopback (Windows especially)"
    **First, the common cause — Npcap without loopback support (Windows).** Re-run the Npcap installer
    from [npcap.com](https://npcap.com/#download) on its own, and this time **tick "Support loopback
    traffic capture"**. Reboot, reopen Wireshark — the `Npcap Loopback Adapter` now appears. Capture
    on it.

    **If it still shows nothing** (Docker Desktop runs containers in a WSL2 VM, and that traffic can
    stay invisible to Npcap): use the **sample capture** below instead. It is a real HTTP login
    capture — open it and do exactly the "Follow HTTP Stream → find the password" step, no live
    capture needed.

    **⬇ Download the sample capture: [`login-http.pcap`](assets/login-http.pcap)**

    Open it in Wireshark (File → Open), right-click the `POST /login` row → **Follow** → **HTTP
    Stream**, and you will see `username=alice&password=alicepw` in clear text — the same result the
    live capture gives.

    **One more independent check that works on every OS:** open your browser's DevTools (F12) →
    **Network** tab → log in → click the `login` request → **Payload** / **Request** tab. You will
    see the username and password in clear text there too. (Note: DevTools shows the browser's own
    copy, so it looks the same over HTTPS — it proves the data is plaintext in the request, but only
    Wireshark proves whether it is *also* readable on the wire. That difference is the whole point of
    Part 3.)

??? bug "'docker compose' complains about nginx/certs not found"
    You skipped Step 1, or ran the commands from the wrong directory. You must be inside the
    `notesapp/` folder for every command in this guide.
