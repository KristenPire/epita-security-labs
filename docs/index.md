# Computer Security — Labs

Hands-on labs for the EPITA Computer Security course. Work in your **pair**, on your own laptop,
inside your NotesApp clone from Class 2.

## Class 5 — HTTPS & password storage (2 h)

Two defenses, two different problems:

- **HTTPS** protects data **on the wire** — between your browser and the server.
- **Password hashing** protects data **at rest** — inside the database.

Missing either one, and the other does not save you. Today you build both.

| Lab | What you do | Time |
|---|---|---|
| [Lab 0 — Set up before class](class5/setup.md) | Install Docker + Wireshark, clone NotesApp | ~15 min, **at home** |
| [Lab 1 — HTTPS for NotesApp](class5/https-lab.md) | Capture a password over HTTP, turn on HTTPS, capture again | ~45 min |
| [Lab 2A — Crack weak hashes](class5/password-crack.md) | Crack an unsalted-MD5 dump with a dictionary attack | ~15 min |
| [Lab 2B — Fix it with Argon2](class5/password-fix.md) | Migrate NotesApp's real login to Argon2, upgrade-on-login | ~20–25 min |

!!! warning "Ethics — only your own targets"
    Every attack in these labs runs **only** on data you generated yourself, from the course's own
    seed data, or on your own NotesApp clone. The techniques are identical to what you would run
    against a real leaked database — that is exactly why they only ever touch targets you are
    allowed to touch. This is your ethics charter, in practice.

## Before you start

You need:

- **Docker** with the Compose plugin (`docker compose version` should work).
- **Wireshark** — free, from [wireshark.org/download](https://www.wireshark.org/download.html).
- Your **NotesApp** clone. If you lost it:

    ```bash
    git clone git@github.com:<your-username>/NoteApp.git notesapp
    cd notesapp
    ```

!!! tip "Work as a pair"
    One person drives the keyboard, the other reads steps out loud and checks the checkpoints — then
    swap. Stuck for more than two minutes? Don't debug in silence — scroll to the **Troubleshooting**
    box at the bottom of the page. Every common blocker is listed there with a fix.

!!! warning "Do [Lab 0 — Setup](class5/setup.md) before class"
    Installing Docker and Wireshark (especially on Windows) can need a reboot. Get the three setup
    checkmarks green at home so you don't lose lab time to installs.
