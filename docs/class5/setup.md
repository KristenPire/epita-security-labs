# Lab 0 — Set up before class

Do this **before** Class 5, not during it. Installing Wireshark on Windows can need a reboot, and
you do not want to spend lab time on it. ~15 minutes, once.

!!! warning "This one really is a before-class step"
    If you arrive with nothing installed, you will lose the first 20 minutes of a 2-hour lab. Get
    the three checkmarks below green at home.

## 1. Docker

Install **Docker Desktop** (Windows / macOS) or Docker Engine (Linux).

```bash
docker compose version
```

!!! success "Checkpoint"
    Prints a version like `Docker Compose version v2.x`. If "command not found", Docker is not
    installed or not running — start Docker Desktop and try again.

## 2. Wireshark (with loopback capture)

Install from [wireshark.org/download](https://www.wireshark.org/download.html).

=== "Windows"

    During install, **let it install Npcap**, and on the Npcap screen **tick "Support loopback
    traffic capture"**. This is the box people miss — without it you cannot capture `localhost`, and
    that is the whole first exercise.

    Already installed Wireshark without that option? Re-run the Npcap installer alone (download from
    [npcap.com](https://npcap.com/#download)), and tick "Support loopback traffic capture" this time.
    Reboot after.

=== "macOS"

    The installer sets up the capture helper (ChmodBPF) for you. If prompted, allow it and reboot.
    Your loopback interface is `Loopback: lo0`.

=== "Linux"

    ```bash
    sudo apt install wireshark      # choose "Yes" to let non-root users capture
    sudo usermod -aG wireshark $USER
    ```

    Log out and back in. Your loopback interface is `lo`.

!!! success "Checkpoint"
    Open Wireshark. In the interface list you can see a **loopback** interface — `lo0` (macOS) / `lo`
    (Linux) / `Adapter for loopback traffic capture` (Windows; older Npcap: `Npcap Loopback Adapter`)
    — and double-clicking it starts a live capture without an error.

## 3. NotesApp

Use your clone from Class 2. Lost it?

```bash
git clone git@github.com:<your-username>/NoteApp.git notesapp
cd notesapp
docker compose up -d
```

!!! success "Checkpoint"
    <http://localhost:8080> shows the NotesApp login page. (Then `docker compose down` until class if
    you like — your data lives in a volume and is kept.)

---

All three green? You are ready. Start with [Lab 1 — HTTPS](https-lab.md) in class.
