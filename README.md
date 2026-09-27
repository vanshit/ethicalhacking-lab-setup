# 🧪 Isolated Ethical Hacking Lab — Setup Guide

![Docker](https://img.shields.io/badge/docker-latest-blue)     ![Kali Linux](https://img.shields.io/badge/kali--linux-rolling-557C94)     ![OWASP Juice Shop](https://img.shields.io/badge/juice--shop-latest-red)     ![Metasploitable](https://img.shields.io/badge/metasploitable-2-orange)

> A fully Dockerised, network-isolated penetration testing lab for hands-on offensive security training.

---

## 📋 Overview

This lab gives every attendee a self-contained environment to practice ethical hacking techniques safely, without touching production networks or the internet.

| Role | Component | Image | Access |
|---|---|---|---|
| 🗡️ Attacker Machine | Kali Linux | [`linuxserver/kali-linux`](https://hub.docker.com/r/linuxserver/kali-linux) | Browser (Web GUI) + Shell |
| 🎯 Victim Machine (OS-level) | Metasploitable2 | [`tleemcjr/metasploitable2`](https://hub.docker.com/r/tleemcjr/metasploitable2) | Shell |
| 🎯 Victim Web App | OWASP Juice Shop | [`bkimminich/juice-shop`](https://hub.docker.com/r/bkimminich/juice-shop) | Browser |

All three containers are **isolated from the host and internet-facing traffic** by attaching them to a dedicated, custom Docker bridge network: **`pentest-lab`**.

```
🔒 pentest-lab (isolated Docker network)

  🗡️  kali-attacker (Attacker)
        │
        ├──attacks──▶ 🎯 metasploitable2 (Victim – OS/Services)
        │
        └──attacks──▶ 🎯 juice-shop (Victim – Web App)
```

> ⚠️ **Safety Note:** These images (Metasploitable2, Juice Shop) are *intentionally vulnerable*. Never expose this network or these containers to the public internet or an untrusted LAN. Keep them on the isolated `pentest-lab` network only, and destroy containers after the exercise.

---

## 🖥️ Installing Docker (Choose Your OS)

Docker must be installed and running **before** you do anything else in this guide. Pick your operating system below.

### 🍎 macOS (Apple Silicon — M1/M2/M3/M4)

| Requirement | Details |
|---|---|
| macOS version | Current release or either of the two previous major versions |
| RAM | 4 GB minimum (8–16 GB recommended for running all 3 containers comfortably) |
| Disk | 10 GB+ free |
| Extra | Rosetta 2 recommended (needed to run x86_64-only images under emulation) |

**Install steps:**

```bash
# Option A — Homebrew (recommended)
brew install --cask docker

# Option B — Manual download
# Visit https://www.docker.com/products/docker-desktop and download
# the "Apple Silicon" build (NOT the Intel chip build)
```

**Step 1 — Open Docker.app**

Once installed, launch it from Applications.

**Step 2 — Install Rosetta 2 if prompted**

Or run manually: `softwareupdate --install-rosetta`

**Step 3 — Wait for Docker to start**

The whale icon in the menu bar will confirm Docker is running.

**Step 4 — (Recommended) Allocate resources**

Go to **Docker Desktop → Settings → Resources** and allocate at least **4 CPUs / 8 GB RAM** so all three lab containers run smoothly.

**Verify installation:**

```bash
docker --version
docker compose version
docker run hello-world
```

> ⚠️ **Apple Silicon architecture note:** `linuxserver/kali-linux` and `bkimminich/juice-shop` publish native **arm64** images, so they run at full speed. `tleemcjr/metasploitable2`, however, is only published for **amd64** — on Apple Silicon it will run under emulation (Rosetta/QEMU). It still works, but expect it to be noticeably slower to boot and to respond to scans/exploits. This guide flags the one command where you may need to add `--platform linux/amd64` explicitly.

---

### 🪟 Windows 10 / 11

| Requirement | Details |
|---|---|
| OS version | Windows 10 64-bit 22H2+ or Windows 11 64-bit 23H2+ (Home, Pro, Enterprise, or Education) |
| RAM | 8 GB minimum |
| CPU | 64-bit with SLAT (Second Level Address Translation) support |
| Extra | Hardware virtualization enabled in BIOS/UEFI, WSL 2 feature enabled |

**Install steps:**

**Step 1 — Enable virtualization in BIOS/UEFI**

This varies by manufacturer (usually labeled `Intel VT-x` or `AMD-V`). Skip this step if it's already enabled.

**Step 2 — Enable WSL 2**

Open PowerShell **as Administrator** and run:

```powershell
wsl --install
```

Restart your PC when prompted.

**Step 3 — Download Docker Desktop for Windows**

Get it from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop).

**Step 4 — Run the installer**

Make sure **"Use WSL 2 instead of Hyper-V"** is checked when prompted.

**Step 5 — Restart your PC**

Restart once installation completes.

**Step 6 — Launch Docker Desktop**

Wait for it to say "Docker Desktop is running."

**Verify installation** (PowerShell or Windows Terminal):

```powershell
docker --version
docker compose version
docker run hello-world
```

> 💡 **Recommended for this lab**
>
> - Run all the `docker` commands in this guide from inside a **WSL 2 Ubuntu terminal**, not PowerShell.
> - All commands here use bash-style line continuation (`\`), which works as-is in WSL 2 / Ubuntu / macOS / Linux terminals — but **not** in PowerShell, which uses a backtick `` ` `` instead.
> - To get a WSL 2 Ubuntu terminal: run `wsl --install -d Ubuntu`, then open **"Ubuntu"** from the Start Menu.
>
> **Prefer to stay in PowerShell?** Then either:
>
> - Run each command as a single line (remove the `\` line breaks), **or**
> - Replace every trailing `\` with a backtick `` ` ``
>
> ⚠️ Also confirm Docker Desktop is set to **"Linux containers"** mode (the default). Right-click the Docker whale icon in the system tray to check — Windows containers cannot run these Linux-based images.

---

### 🐧 Linux (native Docker Engine)

If you're on native Linux, just install Docker Engine + the Compose plugin per your distro (e.g. `curl -fsSL https://get.docker.com | sh`), then add your user to the `docker` group (`sudo usermod -aG docker $USER`) so you don't need `sudo` for every command.

---

## ✅ Prerequisites Checklist

- [ ] Docker Engine installed (v20.10+) and running
- [ ] Docker Compose v2 available (`docker compose`, not the legacy `docker-compose`)
- [ ] At least 4 GB free RAM (8 GB+ recommended) and 10 GB free disk
- [ ] Basic familiarity with terminal / shell commands
- [ ] **(Windows only)** Using a WSL 2 Ubuntu terminal, or adjusted for PowerShell syntax
- [ ] **(Apple Silicon only)** Rosetta 2 installed, aware Metasploitable2 runs emulated

Check your install:

```bash
docker --version
docker compose version
```

---

## 🌐 Step 1 — Create the Isolated Docker Network

Regardless of which setup method you choose (individual containers or Compose), the containers must share the same isolated network.

```bash
docker network create --driver bridge pentest-lab
```

Verify it was created:

```bash
docker network inspect pentest-lab
```

> This network is **not** attached to your host's default bridge and has no external routing configured, keeping lab traffic contained among the three containers.

---

## 🛠️ Setup Method 1 — Individual Container Instances

Use this method if you want to start/stop/inspect each machine independently, or run the lab manually step-by-step for teaching purposes.

> 🪟 **Windows users:** run these commands from a WSL 2 Ubuntu terminal (recommended) so the `\` line continuations work unchanged. See the [Installing Docker](#-installing-docker-choose-your-os) section above if you haven't set this up yet.

### 1️⃣ Kali Linux (Attacker)

```bash
docker run --rm -it \
  --name kali-attacker \
  --hostname hackdoor \
  --network pentest-lab \
  --privileged \
  --shm-size=1gb \
  -p 3001:3001 \
  -e CUSTOM_USER=kali \
  -e PASSWORD=kali \
  lscr.io/linuxserver/kali-linux:latest bash
```

> This drops you straight into an interactive `bash` shell inside the container once it starts. `CUSTOM_USER` / `PASSWORD` also set the login credentials for the web GUI (see [Access](#-how-attendees-access-each-instance) below). `--rm` means the container is removed as soon as you exit the shell — leave this shell open for the duration of the exercise, or drop `--rm -it ... bash` and run detached (`-d`) if you'd rather manage it like the other containers.

### 2️⃣ Metasploitable2 (Victim — OS/Services)

```bash
docker run -d \
  --name metasploitable2 \
  --network pentest-lab \
  --cap-add=NET_RAW \
  --cap-add=NET_ADMIN \
  tleemcjr/metasploitable2
```

> 🍎 **On Apple Silicon (M1/M2/M3/M4):** this image is amd64-only. If Docker complains about a missing manifest or you want to force emulation explicitly, add `--platform linux/amd64` right after `docker run`:
> ```bash
> docker run -d --platform linux/amd64 \
>   --name metasploitable2 \
>   --network pentest-lab \
>   --cap-add=NET_RAW \
>   --cap-add=NET_ADMIN \
>   tleemcjr/metasploitable2
> ```
> It will run fine via Rosetta/QEMU emulation, just slower to boot than on Intel/Windows/Linux hosts.

### 3️⃣ OWASP Juice Shop (Victim — Web App)

```bash
docker run -d \
  --name juice-shop \
  --network pentest-lab \
  -p 3000:3000 \
  bkimminich/juice-shop
```

### Check all containers are running and on the right network

```bash
docker ps
docker network inspect pentest-lab --format '{{range .Containers}}{{.Name}} - {{.IPv4Address}}{{"\n"}}{{end}}'
```

---

## 🚀 Setup Method 2 — Docker Compose (One-Shot Setup)

Use this method to spin up the entire lab — network included — in a single command. Ideal for classrooms/workshops where every attendee needs an identical environment fast.

> ✅ **Cross-platform by design:** the `docker-compose.yml` below already includes the `platform: linux/amd64` fix for Metasploitable2, so it works unchanged on macOS (Intel or Apple Silicon), Windows, and Linux — no edits needed per OS. Windows users should still run `docker compose up -d` from a WSL 2 Ubuntu terminal or PowerShell (both work fine for this single command, since there's no `\` line continuation involved).

### `docker-compose.yml`

```yaml
version: "3.9"

services:
  kali-attacker:
    image: lscr.io/linuxserver/kali-linux:latest
    container_name: kali-attacker
    hostname: hackdoor
    privileged: true
    environment:
      - CUSTOM_USER=kali
      - PASSWORD=kali
    ports:
      - "3001:3001"   # Kali Web GUI (KasmVNC/noVNC)
    shm_size: "1gb"
    command: bash
    tty: true
    stdin_open: true
    networks:
      - pentest-lab
    restart: unless-stopped

  metasploitable2:
    image: tleemcjr/metasploitable2
    container_name: metasploitable2
    platform: linux/amd64   # 🍎 Needed on Apple Silicon; harmless no-op on Intel/Windows/Linux
    cap_add:
      - NET_RAW
      - NET_ADMIN
    networks:
      - pentest-lab
    restart: unless-stopped

  juice-shop:
    image: bkimminich/juice-shop
    container_name: juice-shop
    ports:
      - "3000:3000"   # Juice Shop Web App
    networks:
      - pentest-lab
    restart: unless-stopped

networks:
  pentest-lab:
    driver: bridge
    name: pentest-lab
```

### Bring the lab up

```bash
docker compose up -d
```

### Tear the lab down (stop + remove containers, keep network definition)

```bash
docker compose down
```

### Full teardown (also remove network + volumes)

```bash
docker compose down --volumes --remove-orphans
```

---

## 🔑 How Attendees Access Each Instance

| Machine | Access Type | How to Connect |
|---|---|---|
| 🗡️ **Kali Linux** | 🌐 Browser | `http://localhost:3001` (or host-IP:3001) — opens the Kali desktop via web GUI. Login: `kali` / `kali` |
| 🗡️ **Kali Linux** | 💻 Shell | Attached automatically when you run the `docker run --rm -it ... bash` command. For a second shell into an already-running container: `docker exec -it kali-attacker bash` |
| 🎯 **Metasploitable2** | 💻 Shell | `docker exec -it metasploitable2 /bin/bash` |
| 🎯 **OWASP Juice Shop** | 🌐 Browser | `http://localhost:3000` |

### Quick reference commands

```bash
# Shell into Kali (attacker box) — new session
docker exec -it kali-attacker bash

# Shell into Metasploitable2 (victim box)
docker exec -it metasploitable2 /bin/bash

# Get Kali's internal IP (to target Metasploitable/Juice Shop from inside Kali)
docker network inspect pentest-lab
```

> 💡 **Tip:** Once inside the Kali shell, attendees can resolve the other two machines directly by container name (Docker's embedded DNS on custom bridge networks resolves service/container names automatically):
> ```bash
> ping metasploitable2
> ping juice-shop
> curl http://juice-shop:3000
> ```

---

## 🧭 Suggested Exercise Flow

1. **Recon** — From inside `kali-attacker`, run `nmap -sV metasploitable2` and `nmap -sV juice-shop` to enumerate open ports/services.
2. **Exploit Metasploitable2** — Use `msfconsole` (pre-installed on Kali) to exploit known vulnerable services (e.g., vsftpd backdoor, Samba, etc.).
3. **Attack Juice Shop** — Use Burp Suite / OWASP ZAP (available on Kali) against `http://juice-shop:3000` to work through the [Juice Shop challenge list](https://pwning.owasp-juice.shop/).
4. **Document findings** — Encourage attendees to log each vulnerability, exploitation path, and remediation.

---

## 🧹 Cleanup

When the session ends, always tear the lab down to avoid leaving vulnerable services running:

```bash
# If using individual containers
docker rm -f kali-attacker metasploitable2 juice-shop
docker network rm pentest-lab

# If using Compose
docker compose down --volumes --remove-orphans
```

---

## ⚠️ Responsible Use Reminder

This lab is built entirely from **intentionally vulnerable** software (Metasploitable2, Juice Shop) for **authorized, educational use only**. Do not:
- Expose these containers/ports beyond your isolated training network
- Use techniques learned here against systems you do not own or have explicit written authorization to test

Keep everything confined to the `pentest-lab` Docker network at all times.

---

## 🧩 Optional: Headless Kali (No Browser/GUI Needed)

If attendees don't need a desktop GUI and are comfortable working purely from a terminal, you can use the official **minimal Kali image** instead of `linuxserver/kali-linux`. It's smaller, faster to pull, and gives shell-only access — perfect for CLI-only exercises or lower-resource machines.

> ℹ️ This is a **separate, optional path** — it is not part of Setup Method 1 or the `docker-compose.yml` above. Use it only if you specifically want a lightweight, browser-free Kali.

**Image used:** [`kalilinux/kali-rolling`](https://hub.docker.com/r/kalilinux/kali-rolling)

### Step 1 — Pull the image

```bash
docker pull kalilinux/kali-rolling
```

### Step 2 — Run it and attach it to the lab network

```bash
docker run -it \
  --name kali-headless \
  --network pentest-lab \
  kalilinux/kali-rolling /bin/bash
```

This drops you straight into a root shell inside a bare-bones Kali container, already connected to the same `pentest-lab` network as your Metasploitable2 and Juice Shop targets — so it can reach them just like the GUI version.

### Step 3 — Update package lists

```bash
apt update
```

### Step 4 — Install the tools you need

The `kalilinux/kali-rolling` image ships almost empty by design, so you install tools in groups ("metapackages") based on what the exercise needs:

| Command | What it installs |
|---|---|
| `apt install -y kali-linux-headless` | Kali's default toolset, minus anything GUI-only — the core "headless" base |
| `apt install -y kali-tools-top10` | The 10 most-used pentest tools — **includes Metasploit Framework**, along with nmap, Burp Suite, Hydra, John the Ripper, sqlmap, Wireshark, Aircrack-ng, Hashcat, and CrackMapExec |
| `apt install -y kali-tools-web` | Web app testing tools — sqlmap, Nikto, dirb, gobuster, wfuzz, OWASP ZAP (great for attacking Juice Shop) |
| `apt install -y kali-tools-information-gathering` | Recon/enumeration tools for scanning and fingerprinting targets |

Run them one after another:

```bash
apt install -y kali-linux-headless
apt install -y kali-tools-top10
apt install -y kali-tools-web
apt install -y kali-tools-information-gathering
```

> ✅ **Metasploit Framework** is already included as part of `kali-tools-top10` — no separate install step is needed. Once installed, launch it with:
> ```bash
> msfconsole
> ```

### Verifying it can reach the other lab machines

```bash
ping metasploitable2
ping juice-shop
curl http://juice-shop:3000
```

### Re-attaching to an already-running headless container

If you started the container with `-it` and later detached (or opened a new terminal), get back in with:

```bash
docker exec -it kali-headless /bin/bash
```

### Making the installed tools persistent

By default, anything installed inside a container is lost once the container is removed. Two common ways to avoid reinstalling every time:

- **Keep the container instead of removing it** — just `docker start -ai kali-headless` next time instead of `docker run` again.
- **Build your own image** once tools are installed, so future labs start pre-loaded:
  ```bash
  docker commit kali-headless my-kali-pentest:latest
  ```
  Then next time, run `docker run -it --network pentest-lab my-kali-pentest:latest /bin/bash` instead of pulling and reinstalling from scratch.

---

## 📜 Ethical Usage Disclaimer

### ✅ Acceptable Use Agreement

By setting up and using this lab, every attendee agrees to the following:

- 🎯 **Scope is limited to this lab.** Techniques, tools, and exploits practiced here may only be used against the containers you deployed on the isolated `pentest-lab` network — never against any other system, network, device, or account.
- 🚫 **No unauthorized access, ever.** Do not point any scanner, exploit, or tool from this lab at any system you do not own or do not have explicit, written, prior authorization to test.
- 🔒 **Keep it isolated.** Do not bridge, expose, port-forward, or otherwise connect `pentest-lab` or its containers to the internet or to any production/shared network.
- 🧑‍🎓 **Educational intent only.** This environment exists to teach offensive security concepts defensively — to help attendees understand and *prevent* real-world attacks, not to enable them elsewhere.
- 🗑️ **Clean up after use.** Tear down the lab (see [Cleanup](#-cleanup)) once the exercise or training session ends.
- ✍️ **Individual accountability.** Each attendee is personally responsible for how they use the skills learned in this lab, both during and after the session.

> ⚠️ Trainers/organizers should have attendees explicitly acknowledge (verbally, or via a signed/e-signed form) that they've read and agree to this section **before** lab access is granted.

---

### ⚖️ Legal & Ethical Boundaries

Practicing on this lab's own isolated containers is legal and encouraged. Using the **same techniques against systems without authorization is a criminal offense** in most jurisdictions. In India, this includes (but isn't limited to) the **Information Technology Act, 2000 (IT Act 2000)**:

| Section | Covers |
|---|---|
| **Section 43** | Unauthorized access, damage, or introduction of viruses/malware into a computer system — civil penalty/compensation |
| **Section 66** | Computer-related offences — dishonest or fraudulent acts under Section 43, punishable with imprisonment and/or fine |
| **Section 66C** | Identity theft — fraudulent use of another person's credentials |
| **Section 66D** | Cheating by personation using computer resources |
| **Section 70** | Unauthorized access to a "protected system" (critical/government infrastructure) — more severe penalties |
| **Section 72** | Breach of confidentiality and privacy of data accessed |

> 🌍 **Not India-based?** Equivalent laws exist elsewhere — e.g. the **Computer Fraud and Abuse Act (CFAA)** in the US, the **Computer Misuse Act 1990** in the UK, and similar cybercrime legislation in most countries. Ignorance of local law is not a defense.

> 🧑‍⚖️ This section is for awareness only and is **not legal advice**. Consult a qualified legal professional for guidance specific to your situation or jurisdiction.

---

### 📢 Responsible Disclosure

If, outside this lab, you discover a real vulnerability in a system you do **not** own:

1. 🛑 **Stop.** Do not exploit it further, extract data, or share it publicly.
2. 📩 **Report it privately** to the organization — via their published `security.txt`, a `security@` email address, or their bug bounty platform (HackerOne, Bugcrowd, etc.).
3. ⏳ **Give them time to fix it.** Standard coordinated-disclosure windows are typically **90 days**, though this varies by organization/program.
4. 🤐 **Don't disclose publicly** until the vendor confirms a fix, agrees to a disclosure date, or the agreed window has passed.
5. 🙅 **Never demand payment as a condition of disclosure** — that crosses from "researcher" into "extortion" territory, legally and ethically.

---

### 🏆 Bug Bounty Ethics

If applying these skills through a legitimate bug bounty program:

- 📋 **Stay strictly within the program's defined scope** (domains, IPs, apps) — testing out-of-scope assets is unauthorized access, not research.
- 📖 **Read and follow the program's rules of engagement** before testing (allowed techniques, rate limits, prohibited actions like DoS or social engineering unless explicitly in scope).
- 🧪 **Prove impact minimally.** Demonstrate a vulnerability exists without excessive data access, exfiltration, or damage — a proof-of-concept, not a full exploitation.
- 🔐 **Protect any data you do encounter.** Don't retain, share, or misuse it; report and delete.
- 🗣️ **Report only through official channels**, and don't publicly disclose findings before the program permits it.
- 🤝 **Respect the researcher-organization relationship** — one bad-faith report can damage trust for the entire security research community.
