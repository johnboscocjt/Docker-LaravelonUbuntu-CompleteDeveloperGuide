# 🚑 Docker + Laravel — Survival, Transfer & Deployment Guide

> **Author:** [@johnboscocjt](https://github.com/johnboscocjt)
> **Who this is for:** Developers on limited data · Developers sharing projects between PCs · Anyone deploying Laravel with Docker
> **Covers:** Docker Contexts · Incomplete builds · Why Docker re-downloads things · Offline transfer · Running on another PC · Hot containers · Production deployment

---

## 📑 Table of Contents

1. [🎯 Docker Contexts — The Simple Truth (Read This First)](#-docker-contexts--the-simple-truth-read-this-first)
2. [🐧 Why Docker Downloads Its Own Ubuntu (Even If You Have Ubuntu)](#-why-docker-downloads-its-own-ubuntu-even-if-you-have-ubuntu)
3. [⚠️ Your Build Stopped Mid-Way — What to Do Without Wasting MBs](#️-your-build-stopped-mid-way--what-to-do-without-wasting-mbs)
4. [✂️ Stripping Down Your Project to Save Data](#️-stripping-down-your-project-to-save-data)
5. [🔌 Port Conflict — `bind: address already in use`](#-port-conflict--bind-address-already-in-use)
6. [🚀 Docker Desktop Doesn't Start Automatically](#-docker-desktop-doesnt-start-automatically)
7. [💾 Running Your Laravel Project on Another PC](#-running-your-laravel-project-on-another-pc)
8. [📦 Saving & Transferring Images via Flash Drive (Zero Internet)](#-saving--transferring-images-via-flash-drive-zero-internet)
9. [🔥 Hot Containers — Sharing a Live Laravel App on Your Local Network](#-hot-containers--sharing-a-live-laravel-app-on-your-local-network)
10. [🌍 Deploying Laravel + Docker to a Real Server (Production)](#-deploying-laravel--docker-to-a-real-server-production)
11. [🧰 Pre-Flight Checklist Before Every `sail up`](#-pre-flight-checklist-before-every-sail-up)
12. [🗺️ Quick Reference — All Commands in One Place](#️-quick-reference--all-commands-in-one-place)

---

## 🎯 Docker Contexts — The Simple Truth (Read This First)

> You have already run `docker context use default`. This section explains exactly what that means, why it matters, and the one simple rule you need to follow so you never get confused again.

---

### What Is a Docker Context?

Think of a **context** as a remote control that points at a specific Docker engine. When you run any Docker or Sail command, it gets sent to whichever engine your context is currently pointing at.

You have **two Docker engines** installed on your machine:

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR UBUNTU PC                       │
│                                                         │
│  Engine 1: Docker Engine (the "default" context)        │
│  ─────────────────────────────────────────────          │
│  • Runs as a background system service                  │
│  • Starts automatically on boot                         │
│  • Controlled via terminal only                         │
│  • socket: /var/run/docker.sock                         │
│                                                         │
│  Engine 2: Docker Desktop (the "desktop-linux" context) │
│  ─────────────────────────────────────────────          │
│  • Runs inside its own small virtual machine            │
│  • Only works when Docker Desktop app is open           │
│  • Has its own GUI dashboard                            │
│  • Has its own separate storage for images/containers   │
└─────────────────────────────────────────────────────────┘
```

**The most important thing to understand:**

> These two engines are completely separate from each other. Images you download in one engine do NOT appear in the other. Containers you start in one engine do NOT appear in the other. They each have their own storage.

---

### The One Rule You Need

**Whatever context you pick — stick with it for everything.**

Pick one, use it for all your projects, and never switch mid-project.

```
You already ran:  docker context use default
That means:       You are using Docker Engine (the system service)
Stick with:       Always use "default" from now on
Never switch to:  desktop-linux mid-project
```

---

### What "default" Context Means for You Right Now

Since you ran `docker context use default`, here is exactly what your setup looks like:

```
docker context use default   ← you already did this ✅

Every command you run now goes to:   Docker Engine (the system service)
Your containers live in:             Docker Engine's storage
Your images live in:                 Docker Engine's storage
Docker Desktop GUI shows:            NOTHING (different engine, different storage)
```

**This is completely fine.** Most developers who work mostly in the terminal use `default`. You do not need Docker Desktop at all.

---

### Will Docker Desktop Show My Containers?

No — and that is expected, not a bug.

Since you are using the `default` context, your containers and images live inside Docker Engine's storage. Docker Desktop looks at its own separate engine (`desktop-linux`). It will show an empty container list even though your Laravel project is running perfectly.

```
Your terminal (default context):          Docker Desktop GUI:
─────────────────────────────────         ──────────────────────────
docker ps                                 Containers tab
→ laravel.test   ✅ running               → (empty — different engine)
→ mysql          ✅ running               → nothing here
→ redis          ✅ running               → nothing here
```

**Your app is working. Docker Desktop just cannot see it.** This is normal.

If you want Docker Desktop to show your containers, you would need to switch context to `desktop-linux` — but then you would need Docker Desktop running at all times. Since you are on `default`, you do not.

---

### Checking Which Context You Are On Right Now

```bash
docker context ls
```

Example output:

```
NAME              DESCRIPTION                               DOCKER ENDPOINT
default *         Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux     Docker Desktop                            ...
```

The `*` next to `default` means you are on Docker Engine. That is where your containers and images live.

---

### The Only Time You Should Change Context

Do not change context unless you have a very specific reason. Here are the only valid reasons:

| Situation | What to Do |
|-----------|-----------|
| You want to use the Docker Desktop GUI to see containers | `docker context use desktop-linux` — but then always use Desktop for everything |
| You accidentally switched context mid-project | Switch back to `default` immediately: `docker context use default` |
| You are setting up a completely fresh machine and choosing one to use | Pick `default` if you prefer terminal. Pick `desktop-linux` if you prefer GUI. Then never switch |
| You are deploying to a remote server | Use `default` — servers never have Docker Desktop |

---

### The Golden Rule — One Context, Always

```bash
# ✅ CORRECT — pick one and stay on it
docker context use default        # Terminal-first developer
# or
docker context use desktop-linux  # GUI-first developer

# ❌ WRONG — switching back and forth causes confusion
docker context use desktop-linux
./vendor/bin/sail up -d
docker context use default        # Now sail can't see the containers it just started!
docker ps                         # Shows nothing — they're in the other engine
```

---

### Summary: Your Situation Right Now

You ran `docker context use default`. Here is your complete picture:

```
✅ Context:          default  (Docker Engine — system service)
✅ Auto-starts:      Yes — Docker Engine starts on boot automatically
✅ GUI needed:       No — terminal only, Docker Desktop not required
✅ Your containers:  Live in Docker Engine storage
✅ Your images:      Live in Docker Engine storage
✅ sail up works:    Yes — as long as Docker Engine is running
⚠️  Docker Desktop:  Cannot see your containers (that is normal and fine)
```

**You do not need to change anything.** Just always make sure `docker context ls` shows `default *` before running `sail up`, and you are good.

---

### 🖥️ The `desktop-linux` Context — What It Is and How It Differs

Even though you are on `default`, it is important to understand what `desktop-linux` is so you are never caught off guard when you see it mentioned or when Docker Desktop is involved.

#### What `desktop-linux` Actually Is

When you install Docker Desktop on Linux, it creates its own small virtual machine (VM) running silently in the background. Inside that VM lives a completely separate Docker engine — separate from the one Docker Engine installed on your system.

```
┌──────────────────────────────────────────────────────────────────┐
│                        YOUR UBUNTU PC                            │
│                                                                  │
│  ┌─────────────────────────────────┐                            │
│  │   Docker Engine (system)        │  ← "default" context       │
│  │   /var/run/docker.sock          │                            │
│  │   Starts on boot automatically  │                            │
│  │   No GUI — terminal only        │                            │
│  └─────────────────────────────────┘                            │
│                                                                  │
│  ┌─────────────────────────────────┐                            │
│  │   Docker Desktop VM             │  ← "desktop-linux" context │
│  │   Its own separate engine       │                            │
│  │   Only runs when Desktop is open│                            │
│  │   Has the GUI dashboard         │                            │
│  │   Has its own images + storage  │                            │
│  └─────────────────────────────────┘                            │
│                                                                  │
│  These two engines NEVER share images, containers, or volumes.  │
└──────────────────────────────────────────────────────────────────┘
```

#### How `desktop-linux` Behaves Day-to-Day

| Behaviour | `desktop-linux` |
|-----------|----------------|
| Starts automatically on boot | ❌ No — you must open Docker Desktop first |
| Needs an app to be open | ✅ Yes — Docker Desktop must be running |
| Has a visual GUI dashboard | ✅ Yes — containers, logs, stats, images all visible |
| Works from the terminal | ✅ Yes — but only when Desktop is open |
| Shares images with `default` | ❌ No — completely separate storage |
| Good for limited data users | ⚠️ Adds ~1 GB just for installation |

#### When `desktop-linux` Makes Sense

Use `desktop-linux` if you prefer clicking buttons over typing commands. The Docker Desktop GUI gives you:

- Live container logs in a browser-like interface
- One-click start/stop/restart for any container
- A visual breakdown of disk and memory usage
- A built-in terminal inside any container (no `docker exec` needed)
- Image management with sizes shown clearly

But if you are comfortable in the terminal, `default` gives you everything faster with no overhead.

---

### 📊 Full Comparison: `default` vs `desktop-linux` vs Using Both

This table covers every angle so you can confidently know what each context does, what it costs, and when to use it.

| | `default` (Docker Engine) | `desktop-linux` (Docker Desktop) |
|--|--------------------------|----------------------------------|
| **What it is** | The core Docker daemon running as a Linux system service | A separate Docker engine running inside Docker Desktop's own VM |
| **How to activate** | `docker context use default` | `docker context use desktop-linux` |
| **Starts on boot** | ✅ Yes — automatic, always ready | ❌ No — you must open Docker Desktop first |
| **Requires an app open** | ❌ No — runs silently in background | ✅ Yes — Docker Desktop must be running |
| **Has a GUI** | ❌ No — terminal only | ✅ Yes — full visual dashboard |
| **Terminal commands work** | ✅ Yes — all docker/sail commands | ✅ Yes — but only while Desktop is open |
| **Image storage location** | `/var/lib/docker/` (host filesystem) | Inside Docker Desktop's VM disk image |
| **Shares images with the other** | ❌ No | ❌ No |
| **Shares containers with the other** | ❌ No | ❌ No |
| **Installation size** | ~100–150 MB | ~850 MB–1.1 GB |
| **Good for limited data** | ✅ Yes | ⚠️ Higher cost to install |
| **Auto-start on boot** | `sudo systemctl enable docker` (already done by default) | `systemctl --user enable docker-desktop` |
| **Best for** | Terminal-first developers, servers, CI/CD | Visual learners, beginners, teams using GUI |
| **Sail works with it** | ✅ Yes | ✅ Yes — when Desktop is open |
| **`docker ps` shows containers** | Only containers started under `default` | Only containers started under `desktop-linux` |

---

### 🔴 The Biggest Trap: Starting in One Context, Checking in Another

This is the most common source of confusion. Here is a real example of what goes wrong:

```bash
# You start your project on desktop-linux
docker context use desktop-linux
./vendor/bin/sail up -d
# ✅ Laravel is running

# Later, you switch context (maybe by mistake or following a guide)
docker context use default

# Now you check if containers are running
docker ps
# ❌ Shows nothing — they are in desktop-linux's engine, not default's

# You panic and try to start again
./vendor/bin/sail up -d
# Docker pulls images again from scratch — because default's storage is empty!
# ❌ Wastes hundreds of MB re-downloading what you already have
```

**The fix is always the same:** check your context, switch back to where you started, and your containers will reappear:

```bash
docker context ls                    # see which is active
docker context use desktop-linux     # switch back to where containers live
docker ps                            # ✅ containers appear again — nothing was lost
```

---

### 🟢 Choosing the Right Context for Your Setup — Decision Guide

Answer these questions to know which context is right for you:

```
Do you have Docker Desktop installed?
│
├── No  →  Use "default". Only option available. Done.
│
└── Yes →  Do you want the visual GUI?
           │
           ├── No, I prefer terminal
           │   └──  Use "default"
           │        • Lighter, auto-starts on boot
           │        • Docker Desktop not needed at all
           │        • sudo systemctl enable docker  (already done)
           │
           └── Yes, I want the GUI dashboard
               └──  Use "desktop-linux"
                    • Open Docker Desktop before sail up
                    • systemctl --user enable docker-desktop  (to auto-start)
                    • All containers/images visible in the GUI
```

**You already chose `default`. That is the right choice for a terminal developer on limited data.**

---

### What Each Context Looks Like in Practice

#### `default` Context — Day-to-Day

```bash
# Boot your PC → Docker Engine is already running (auto-start)
# Open terminal → start working immediately

docker context ls
# NAME        DESCRIPTION              DOCKER ENDPOINT
# default *   DOCKER_HOST based config unix:///var/run/docker.sock
# desktop-linux  Docker Desktop        ...

cd my-app
./vendor/bin/sail up -d         # works immediately, no app to open
sail artisan migrate            # runs inside container
# open http://localhost         # ✅ app is live

sail down                       # stop for the day
# Docker Engine keeps running in background, 0 resources used when idle
```

#### `desktop-linux` Context — Day-to-Day

```bash
# Boot your PC → open Docker Desktop app from app menu
# Wait 30–60 seconds for whale 🐳 icon to appear in system tray

docker context use desktop-linux

cd my-app
./vendor/bin/sail up -d         # works once Desktop is open
# ✅ containers appear in Docker Desktop GUI
# click a container → see logs, stats, open terminal inside it

sail down                       # stop for the day
# close Docker Desktop app or leave it open
```

---

### The One Thing Both Contexts Share

Both contexts use the same `docker` and `sail` CLI commands — the syntax never changes. The only difference is **where** those commands are sent (which engine processes them).

```bash
# These commands work identically in BOTH contexts
# Only the destination engine changes based on your active context

docker ps
docker images
docker logs container-name
./vendor/bin/sail up -d
sail artisan migrate
sail down
```

This means you can learn everything once and it applies to both contexts — you just need to stay consistent about which one you are using.

---

### Final Word on Contexts

```
Your context right now:  default ✅
What that means:         You are using Docker Engine
What to do:              Nothing — it is already correct
One command to confirm:  docker context ls   (look for the * next to default)
Rule to follow:          Never switch context unless you have a clear reason
```

If `docker context ls` ever shows the `*` on a context you did not intend, just run `docker context use default` to get back to your setup.

---

### 🔄 How to Switch Context Safely — Without Losing Anything

> You said you want to switch to `desktop-linux` so you can use the Docker Desktop GUI. This section is the exact step-by-step process to do it cleanly, without breaking anything, losing any data, or wasting MBs.

The most important thing to understand before switching:

> **Switching context does NOT delete your containers, images, or data.** Everything you have in `default` stays exactly where it is. You are just moving your remote control to point at a different engine. The old engine and everything in it is still sitting there, untouched, waiting.

---

#### Before You Switch — Understand What Will Happen

```
BEFORE switch:                        AFTER switch to desktop-linux:
──────────────────────────────────    ──────────────────────────────────
default engine:                       default engine:
  └─ laravel images      ✅ exist       └─ laravel images      ✅ still exist
  └─ mysql images        ✅ exist       └─ mysql images        ✅ still exist
  └─ your containers     ✅ running     └─ your containers     ⏸ still there
                                                                 (but not visible
                                                                  from new context)

desktop-linux engine:                 desktop-linux engine:
  └─ (empty)                            └─ (still empty — nothing transferred)
                                        └─ Docker Desktop GUI shows nothing yet
                                        └─ sail up here = fresh download needed ⚠️
```

This means: **switching context alone does NOT move your images to the other engine.** If you switch to `desktop-linux` and run `sail up`, Docker Desktop's engine has no images cached — it will download everything again.

**To avoid this, you must transfer your images first.** The steps below walk you through this exactly.

---

#### The Safe Switch — Step by Step

##### Step 1 — Stop everything cleanly in your current context

Never switch context while containers are running. Stop them first:

```bash
# Make sure you are on default (your current context)
docker context use default

# Go to your project and stop containers cleanly
cd ~/daily-task-tracker-laravel
./vendor/bin/sail down

# Confirm nothing is running
docker ps
# Should show: empty list (no containers)
```

##### Step 2 — Find and note your image names

```bash
docker images
```

```
REPOSITORY                    TAG       IMAGE ID       SIZE
laravelsail/php84-composer    latest    a1b2c3d4     2.01GB
mysql                         8.0       b2c3d4e5      587MB
redis                         alpine    c3d4e5f6       41MB
mailpit/mailpit               latest    d4e5f6a7       52MB
```

Write these down or keep this terminal open — you will need the names in Step 5.

##### Step 3 — Export your images to tar files

This saves your images from Docker Engine's storage into portable files:

```bash
# Create a folder to hold the exported images
mkdir ~/docker-transfer

# Export all your project images into one file
docker save -o ~/docker-transfer/laravel-images.tar \
  laravelsail/php84-composer:latest \
  mysql:8.0 \
  redis:alpine \
  mailpit/mailpit:latest

# Check the file was created
ls -lh ~/docker-transfer/
# Expect: laravel-images.tar   ~2.8–3.2 GB
```

> ☕ This takes a few minutes — it is packaging ~3 GB of images into a single file. No internet used.

##### Step 4 — Start Docker Desktop

```bash
systemctl --user start docker-desktop
# Wait until the 🐳 whale icon appears in your system tray (30–60 seconds)
```

Or open it from your application menu — search **"Docker Desktop"** and click it.

##### Step 5 — Switch context to `desktop-linux`

```bash
docker context use desktop-linux

# Confirm the switch
docker context ls
# NAME              DESCRIPTION       DOCKER ENDPOINT
# default           ...               unix:///var/run/docker.sock
# desktop-linux *   Docker Desktop    ...    ← * is now here ✅
```

##### Step 6 — Load your images into Docker Desktop's engine

```bash
# Import the tar file into the desktop-linux engine
docker load -i ~/docker-transfer/laravel-images.tar
```

You will see each image being loaded:

```
Loaded image: laravelsail/php84-composer:latest
Loaded image: mysql:8.0
Loaded image: redis:alpine
Loaded image: mailpit/mailpit:latest
```

##### Step 7 — Verify images are in Docker Desktop

```bash
docker images
# Should show all your images — same ones as before
```

Also check Docker Desktop GUI — click the **Images** tab and you should see them listed there.

##### Step 8 — Start your project in the new context

```bash
cd ~/daily-task-tracker-laravel
./vendor/bin/sail up -d

# 0 MB downloaded — images are already loaded
# ✅ Containers appear in Docker Desktop GUI
```

Open your browser: `http://localhost` — your app runs exactly as before.

---

#### After the Switch — Your New Daily Workflow

From now on, every time you use your project:

```bash
# 1. Open Docker Desktop first (or enable auto-start — see below)
systemctl --user start docker-desktop

# 2. Make sure context is correct
docker context use desktop-linux

# 3. Start your project
cd ~/daily-task-tracker-laravel
./vendor/bin/sail up -d

# 4. Work normally — open http://localhost
# 5. Stop when done
sail down
```

**To avoid step 1 every time, enable Docker Desktop auto-start:**

```bash
systemctl --user enable docker-desktop
# From now on Docker Desktop starts when you log in — no manual step needed
```

---

#### Clean Up After Switching (Optional — Frees Disk Space)

Once you have confirmed everything works under `desktop-linux`, the images still sitting in Docker Engine's storage (`default`) are now duplicates. You can free that disk space:

```bash
# Switch temporarily to default to clean it up
docker context use default

# Check what's there
docker images

# Remove the images you no longer need in this engine
docker image prune -a
# or remove specific ones:
docker rmi laravelsail/php84-composer:latest mysql:8.0 redis:alpine

# Switch back to desktop-linux (your new home)
docker context use desktop-linux

# Confirm you are back
docker context ls
# desktop-linux *   ← correct
```

> ⚠️ Only clean up `default` after you have confirmed 100% that everything works in `desktop-linux`. Once deleted from `default`, recovering those images means either re-downloading or loading from your tar file again.

---

#### What If You Want to Switch Back to `default` Later?

The process is exactly the same in reverse:

```bash
# Step 1 — Stop containers in desktop-linux
docker context use desktop-linux
sail down

# Step 2 — Export images from desktop-linux
docker save -o ~/docker-transfer/laravel-images-v2.tar \
  laravelsail/php84-composer:latest \
  mysql:8.0 \
  redis:alpine

# Step 3 — Switch context
docker context use default

# Step 4 — Load images into Docker Engine
docker load -i ~/docker-transfer/laravel-images-v2.tar

# Step 5 — Start your project
sail up -d
# 0 MB downloaded ✅
```

---

#### When Is It OK to Switch Without Transferring Images?

You only need to transfer images if you want to avoid re-downloading them. There are two situations where skipping the transfer is acceptable:

| Situation | Transfer needed? | Why |
|-----------|-----------------|-----|
| You have a good internet connection and don't mind re-downloading | ❌ No | Just switch context and let Docker re-pull — it takes time but costs data |
| The other engine already has the images cached from a previous project | ❌ No | Docker reuses them — 0 MB downloaded |
| You are on limited data and images are not cached in the target engine | ✅ Yes | Without transfer, full re-download (~2.5–3 GB) |
| You are switching permanently and want to free disk space after | ✅ Yes | Transfer first, verify, then clean up the old engine |

---

#### Summary: Safe Switch Checklist

```
SWITCHING FROM default → desktop-linux:

□ 1. sail down                              (stop containers cleanly)
□ 2. docker images                          (note your image names)
□ 3. docker save -o ~/transfer.tar [images] (export images — no internet)
□ 4. Open Docker Desktop                    (start the Desktop app)
□ 5. docker context use desktop-linux       (switch context)
□ 6. docker load -i ~/transfer.tar          (import images — no internet)
□ 7. docker images                          (verify they loaded)
□ 8. sail up -d                             (start project — 0 MB downloaded)
□ 9. Test http://localhost                  (confirm app works)
□ 10. (Optional) Clean up default engine    (free duplicate disk space)
□ 11. systemctl --user enable docker-desktop (auto-start for next login)
```

---

---

## 🐧 Why Docker Downloads Its Own Ubuntu (Even If You Have Ubuntu)

When you run `./vendor/bin/sail up` for the first time, Docker reads a file called `Dockerfile`. The very first line inside it is:

```dockerfile
FROM ubuntu:24.04
```

This tells Docker: *"Start by downloading a fresh Ubuntu 24.04 and build everything on top of it."* Docker obeys this instruction literally — **it downloads Ubuntu even if your host computer is already running Ubuntu 24.04.**

### Why Can't Docker Just Use My Existing Ubuntu?

This is intentional. Docker containers are completely sealed from your host system. They cannot see your installed packages, your config files, your PHP version, your MySQL — nothing. This is called **isolation**, and it is Docker's most important feature.

| Reason | What It Means For You |
|--------|----------------------|
| **Isolation** | The container has its own filesystem. It cannot break your main system |
| **Consistency** | Your project runs identically on your PC, your teammate's Mac, a Windows laptop, a cloud server |
| **Self-contained** | The container brings every library it needs — nothing is borrowed from the host |

### The Data Impact of This Isolation

Because Docker cannot use your host packages, it downloads duplicate copies of everything:

```
Your Host Ubuntu (what you already have):        Container Ubuntu (what Docker downloads):
─────────────────────────────────────────        ─────────────────────────────────────────
✅ libxml2 installed                             ⬇️  Downloads libxml2 again  (~3 MB)
✅ curl installed                                ⬇️  Downloads curl again     (~2 MB)
✅ gnupg installed                               ⬇️  Downloads gnupg again    (~1 MB)
✅ ca-certificates installed                     ⬇️  Downloads ca-certs again (~1 MB)
✅ Node.js installed                             ⬇️  Downloads Node.js again  (~30 MB)
... and so on for every package the container needs
```

**Total duplicate downloads: ~50–150 MB** for packages your system already has. This is unavoidable with Docker's isolation model.

### The Good News: It Only Happens Once

Once the build finishes, Docker **caches every single layer** of that image on your disk. Every future `sail up` — even after rebooting — reuses the cache instantly. **0 MB re-downloaded.**

```
First sail up:   Downloads ubuntu:24.04 + all packages  →  ~1.5–2.0 GB
Second sail up:  Reads from cache                        →  0 MB
Third sail up:   Reads from cache                        →  0 MB
```

---

## ⚠️ Your Build Stopped Mid-Way — What to Do Without Wasting MBs

### Recognising a Stopped Build

If you see any of these errors, your download was interrupted:

```
short read: expected 10736047 bytes but got 9549244: unexpected EOF
```
```
net/http: TLS handshake timeout
```
```
failed to resolve source metadata for docker.io/library/ubuntu:24.04
```
```
failed to do request: Head "https://registry-1.docker.io/...": i/o timeout
```

All of these mean the same thing: **your internet connection dropped while Docker was pulling a large layer.**

### What Is Already Saved?

Docker downloads images in **layers** — like a stack of pancakes, each one building on the last. Every layer that fully completed before the dropout is safely saved to your disk.

```
Layer 1  ██████████ ✅ Fully downloaded — saved to disk (0 MB to redo)
Layer 2  ██████████ ✅ Fully downloaded — saved to disk (0 MB to redo)
Layer 3  ██████████ ✅ Fully downloaded — saved to disk (0 MB to redo)
Layer 4  ██████████ ✅ Fully downloaded — saved to disk (0 MB to redo)
Layer 5  █████░░░░░ ❌ Cut off mid-download — only THIS layer re-downloads
Layer 6  ░░░░░░░░░░ ⏳ Never started
Layer 7  ░░░░░░░░░░ ⏳ Never started
```

**You do NOT lose everything.** Only the interrupted layer and everything after it will re-download.

### How to Resume Correctly (Cheapest Option)

Simply run the same command again. Docker automatically detects cached layers and skips them:

```bash
cd daily-task-tracker-laravel
./vendor/bin/sail up
```

You will see output like this:

```
=> CACHED [1/12] FROM ubuntu:24.04                   ← 0 MB, already done
=> CACHED [2/12] RUN apt-get update                  ← 0 MB, already done
=> CACHED [3/12] RUN apt-get install -y curl git      ← 0 MB, already done
=> CACHED [4/12] RUN install php extensions           ← 0 MB, already done
=> [5/12] RUN curl -sLS https://deb.nodesource.com   ← resumes HERE only
```

Only the failed layer and everything after it downloads fresh. Everything before it is free.

### ❌ What NOT to Do

```bash
# NEVER run this after a dropout — it throws away ALL cached layers
# and forces a 1.5–2 GB full re-download from scratch
sail build --no-cache

# NEVER run this — deletes all your images
# Your next sail up re-downloads everything (2–4 GB)
docker system prune -a
```

### If Your Connection Keeps Dropping

If your connection is too unstable to finish the build in one go, use this approach to pull in smaller chunks that are easier to complete:

```bash
# Step 1 — Pull the base Ubuntu image first (smaller, ~80 MB, easier to complete)
docker pull ubuntu:24.04

# Step 2 — Pull Sail image separately (Docker skips ubuntu:24.04 since it's cached)
docker pull laravelsail/php84-composer:latest

# Step 3 — Pull database and other service images separately
docker pull mysql:8.0
docker pull redis:alpine

# Step 4 — Now start your project — everything is cached, build is instant
./vendor/bin/sail up -d
```

---

## ✂️ Stripping Down Your Project to Save Data

If you are very low on data, you can remove services you don't actually need. This prevents Docker from downloading hundreds of megabytes of images you won't use.

### Which Services Can You Remove?

Open `docker-compose.yml` in your project folder. You will see blocks like this:

```yaml
selenium:
    image: 'selenium/standalone-chromium'
    ...

meilisearch:
    image: 'getmeili/meilisearch:latest'
    ...

mailpit:
    image: 'axllent/mailpit:latest'
    ...

redis:
    image: 'redis:alpine'
    ...

mysql:
    image: 'mysql:8.0'
    ...
```

**What each service costs and when you actually need it:**

| Service | Image Size | What It Does | Remove If... |
|---------|-----------|-------------|--------------|
| `selenium` | ~700 MB–1.0 GB | Automated browser testing | You are not running browser tests |
| `meilisearch` | ~100–150 MB | Full-text search | You are not using Laravel Scout |
| `mailpit` | ~50 MB | View emails locally | You are not testing email sending |
| `redis` | ~35 MB | Cache and queues | You are not using queues or cache |
| `mysql` | ~550 MB | Database server | You switch to SQLite (see below) |

**To remove a service**, delete its entire block from `docker-compose.yml`. Also remove it from the `depends_on` section of the `laravel.test` service if it appears there.

---

### Switch to SQLite and Save ~550 MB

SQLite is a file-based database — no server, no container, no download. It is perfect for personal projects, prototypes, and solo development.

**Step 1 — Edit your `.env` file:**

```env
# Before:
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password

# After:
DB_CONNECTION=sqlite
# Comment out or delete the DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD lines
```

**Step 2 — Create the SQLite database file:**

```bash
touch database/database.sqlite
```

**Step 3 — Delete the `mysql` block from `docker-compose.yml`.**

**Step 4 — Start your project:**

```bash
./vendor/bin/sail up -d
sail artisan migrate
```

> 💾 **Data saved: ~550 MB.** The MySQL image is never pulled. Your database is a single file in your project folder.

> ⚠️ SQLite is great for development but not recommended for production with multiple concurrent users. Switch back to MySQL when deploying.

---

## 🔌 Port Conflict — `bind: address already in use`

### What Is Happening

You will see this error when another program on your computer is already using the same port Docker wants:

```
Error starting userland proxy: listen tcp4 0.0.0.0:3306: bind: address already in use
```

The most common cause: you have MySQL installed directly on your Ubuntu system, and it is running in the background on port 3306. Docker's MySQL container also wants port 3306.

```
Host MySQL service  →  listening on port 3306  ✅
Docker MySQL        →  tries to use port 3306  ❌  CONFLICT
```

### Fix Option A — Stop Your Host MySQL (Recommended)

```bash
# For MySQL
sudo service mysql stop

# For MariaDB
sudo service mariadb stop

# Then start your project
./vendor/bin/sail up -d
```

When you stop Sail later, you can restart your host MySQL:

```bash
sudo service mysql start
```

### Fix Option B — Change Docker's External Port

If you need both running at the same time, change the port Docker exposes to your host:

**In your `.env` file:**

```env
# Change this line
FORWARD_DB_PORT=3306

# To this (or any unused port number)
FORWARD_DB_PORT=3307
```

Then restart:

```bash
./vendor/bin/sail down
./vendor/bin/sail up -d
```

> ✅ Your Laravel app still connects to MySQL normally — internally it uses the service name `mysql`, not a port. Only external tools like TablePlus or DBeaver need to use port 3307 now.

### Other Common Port Conflicts

| Port | Service | How to Free It |
|------|---------|---------------|
| 80 | Nginx/Apache on host | `sudo service nginx stop` or `sudo service apache2 stop` |
| 3306 | MySQL/MariaDB on host | `sudo service mysql stop` |
| 6379 | Redis on host | `sudo service redis stop` |
| 8025 | Another Mailpit or mail server | Change `FORWARD_MAILPIT_PORT` in `.env` |

**To find what is using a specific port:**

```bash
sudo lsof -i :3306
# or
sudo ss -tulnp | grep 3306
```

---

## 🚀 Docker Desktop Doesn't Start Automatically

### Why This Happens

Docker Desktop on Linux runs inside a user-managed virtual machine. Unlike Docker Engine (which runs as a system background service), Docker Desktop **only runs when you open it.** If you try `sail up` before Docker Desktop is open, you get:

```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock.
Is the docker daemon running?
```

### Fix A — Start Docker Desktop Manually Each Time

```bash
# From terminal
systemctl --user start docker-desktop

# Wait 30–60 seconds for the whale 🐳 icon to appear in your system tray
# Then run your project
cd my-app && ./vendor/bin/sail up -d
```

Or open it from your application menu, search **"Docker Desktop"** and click it.

### Fix B — Auto-Start Docker Desktop on Login

Run this once:

```bash
systemctl --user enable docker-desktop
```

Docker Desktop now starts automatically every time you log in. No manual steps before `sail up`.

To disable auto-start:

```bash
systemctl --user disable docker-desktop
```

### Fix C — Use Docker Engine Instead (Truly Auto-Starts on Boot)

Docker Engine runs as a proper system service — it starts on boot without you doing anything:

```bash
# Enable Docker Engine to auto-start
sudo systemctl enable docker
sudo systemctl start docker

# Switch your context to use Docker Engine
docker context use default

# Now sail up works any time, even right after booting
./vendor/bin/sail up -d
```

> ⚠️ If you switch to `default` context, containers you created under `desktop-linux` context won't appear. They are not lost — switch context back to see them.

### Always Check Your Context

After starting Docker Desktop, make sure the context is correct before running Sail:

```bash
# See all contexts and which is active (marked with *)
docker context ls

# Switch to Docker Desktop's context
docker context use desktop-linux

# Now Sail connects to the right daemon
./vendor/bin/sail up -d
```

---

## 💾 Running Your Laravel Project on Another PC

There are several ways to move your project to another computer. Each has a different data cost and complexity. Choose the one that fits your situation.

---

### Method 1 — Git Clone + Fresh Pull (Both PCs Have Internet)

**Best for:** Teammates or your own second machine with a decent connection.

**On the first PC — push to GitHub:**

```bash
cd my-app
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/yourname/my-app.git
git push -u origin main
```

**On the second PC — clone and start:**

```bash
git clone https://github.com/yourname/my-app.git
cd my-app
cp .env.example .env
./vendor/bin/sail up -d
sail artisan key:generate
sail artisan migrate --seed
```

**Data cost on second PC:**

| What downloads | Size |
|---------------|------|
| Git repo (code only) | ~5–50 MB (depends on your project) |
| Docker images (if not cached) | ~2.0–3.0 GB |
| Composer packages | ~50–80 MB |
| npm packages | ~150–300 MB |

> 💡 If the second PC has already worked on any Docker Laravel project before, the images are already cached — only the git clone and packages download.

---

### Method 2 — Flash Drive Transfer (No Internet on Second PC)

**Best for:** No internet on the second machine, or very limited data.

This is the only method that truly requires **0 MB of internet** on the second machine. Full details in the next section.

---

### Method 3 — Same Network, Copy Over SSH/SCP

**Best for:** Two computers on the same Wi-Fi or LAN.

**Save images on the first PC:**

```bash
docker save -o ~/laravel_images.tar laravelsail/php84-composer:latest mysql:8.0 redis:alpine
```

**Transfer everything to the second PC over the network:**

```bash
# Copy the images file
scp ~/laravel_images.tar username@secondpc-ip:~/

# Copy the project folder (including vendor/)
scp -r ~/my-app username@secondpc-ip:~/
```

**On the second PC:**

```bash
docker load -i ~/laravel_images.tar
cd ~/my-app
cp .env.example .env
./vendor/bin/sail up -d
sail artisan key:generate
sail artisan migrate
```

---

### What the Second PC Always Needs

Regardless of which method you use, the second computer must have:

- [ ] Docker Engine installed (or Docker Desktop)
- [ ] The `sail` alias set up (`echo "alias sail='sh vendor/bin/sail'" >> ~/.bashrc && source ~/.bashrc`)
- [ ] A `.env` file (copy from `.env.example` and fill in values)
- [ ] An app key generated (`sail artisan key:generate`)
- [ ] Migrations run (`sail artisan migrate`)

---

## 📦 Saving & Transferring Images via Flash Drive (Zero Internet)

### Why Copying Just the Code Doesn't Work

If you copy only your project folder to a flash drive and run it on another computer, Docker will try to download everything from the internet again:

```
Second PC sees:  FROM ubuntu:24.04  →  goes to internet  →  downloads ~1.8 GB
Second PC sees:  mysql:8.0          →  goes to internet  →  downloads ~550 MB
...
Total re-download on second PC: ~2.5–3.5 GB
```

The `Dockerfile` is just a recipe — it tells Docker what to download, not what is already on the flash drive.

### The Correct Method: Export Images as Files

**On the FIRST computer (with images already downloaded):**

**Step 1 — List all your images:**

```bash
docker images
```

```
REPOSITORY                    TAG       IMAGE ID       SIZE
laravelsail/php84-composer    latest    a1b2c3d4     2.01GB
mysql                         8.0       b2c3d4e5     587MB
redis                         alpine    c3d4e5f6     41MB
mailpit/mailpit               latest    d4e5f6a7     52MB
```

**Step 2 — Save all needed images into one file:**

```bash
# Save all project images into a single tar file
docker save -o /media/flashdrive/laravel_images.tar \
  laravelsail/php84-composer:latest \
  mysql:8.0 \
  redis:alpine \
  mailpit/mailpit:latest

# Check the file size
ls -lh /media/flashdrive/laravel_images.tar
# Expect: 2.5–3.0 GB for a full Sail stack
```

**Step 3 — Copy your project folder (include vendor/):**

```bash
# Standard practice excludes vendor/ via .gitignore
# But for offline transfer you MUST include it
# Composer cannot run without internet to download packages

cp -r ~/my-app /media/flashdrive/my-app

# Verify vendor/ is included
ls /media/flashdrive/my-app/vendor/
```

**Step 4 — Copy your `.env` file manually if it exists:**

```bash
cp ~/my-app/.env /media/flashdrive/my-app/.env
```

> `.env` is excluded from git for security reasons — but for a personal transfer to your own second machine it is fine to copy it directly.

---

**On the SECOND computer (no internet needed):**

**Step 1 — Make sure Docker is installed** (Docker Engine or Desktop — installation requires internet, do this before the transfer if possible).

**Step 2 — Load the images from the flash drive:**

```bash
docker load -i /media/flashdrive/laravel_images.tar
```

You will see each image being loaded:

```
Loaded image: laravelsail/php84-composer:latest
Loaded image: mysql:8.0
Loaded image: redis:alpine
Loaded image: mailpit/mailpit:latest
```

**Step 3 — Copy the project to your home folder:**

```bash
cp -r /media/flashdrive/my-app ~/my-app
cd ~/my-app
```

**Step 4 — Start the project — zero internet used:**

```bash
./vendor/bin/sail up -d
# Docker sees all images already loaded — downloads nothing

sail artisan key:generate    # if .env is fresh
sail artisan migrate
```

**Step 5 — Open in browser:**

```
http://localhost
```

---

### Transfer Summary Table

| What You Transfer | Internet on 2nd PC | Data Used on 2nd PC | Works Offline? |
|------------------|-------------------|--------------------|--------------:|
| Code files only | Required | ~2.5–3.5 GB | ❌ No |
| Code + vendor/ | Required | ~2.0–3.0 GB | ❌ No |
| Code + vendor/ + `.tar` images | Not needed | **0 MB** | ✅ Yes |
| Docker image already on 2nd PC | Not needed | 0 MB | ✅ Yes |

---

### Flash Drive Space Needed

| What to Copy | Approximate Size |
|-------------|-----------------|
| `laravel_images.tar` (Sail + MySQL + Redis + Mailpit) | ~2.8–3.2 GB |
| Project code + vendor/ | ~200–500 MB |
| **Total flash drive space needed** | **~3.0–3.7 GB** |

> Use a flash drive of at least **8 GB** to be comfortable.

---

## 🔥 Hot Containers — Sharing a Live Laravel App on Your Local Network

> "Hot container" means your Laravel app is running in Docker on your PC and other people on the same Wi-Fi or LAN can open it in their browser — without installing anything.

---

### Step 1 — Find Your Local IP Address

```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
# or
hostname -I
```

You will see something like `192.168.1.105`. That is your machine's local IP.

### Step 2 — Make Sure Your App Listens on All Interfaces

By default Sail exposes port 80 on all interfaces (`0.0.0.0:80`). Confirm in `docker-compose.yml`:

```yaml
laravel.test:
    ports:
        - '${APP_PORT:-80}:80'
        # This 0.0.0.0 binding is the default — other devices can reach it
```

### Step 3 — Start Your Project

```bash
./vendor/bin/sail up -d
```

### Step 4 — Other Devices Connect

Anyone on the same network opens their browser and goes to:

```
http://192.168.1.105        # replace with your actual IP
http://192.168.1.105:8080   # if you use a custom port
```

That's it. They see your Laravel app live — no Docker, no installation needed on their device.

---

### Making Hot Sharing Work With HTTPS (Optional)

If your app requires HTTPS (e.g., for camera access, service workers), use a tool like `mkcert` to create a local certificate:

```bash
# Install mkcert
sudo apt install libnss3-tools
wget -O mkcert https://github.com/FiloSottile/mkcert/releases/latest/download/mkcert-v1.4.4-linux-amd64
chmod +x mkcert && sudo mv mkcert /usr/local/bin/

# Create a local CA and certificate
mkcert -install
mkcert localhost 127.0.0.1 192.168.1.105

# Place the generated cert files in your project and configure nginx to use them
```

---

### Exposing to the Internet (Temporary — Using a Tunnel)

To share with someone outside your network (no server needed), use `ngrok` or `expose`:

**Using ngrok:**

```bash
# Install ngrok
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok

# Start your app first
./vendor/bin/sail up -d

# Create a public tunnel
ngrok http 80
```

ngrok gives you a public URL like `https://abc123.ngrok.io` — anyone on the internet can open your local Laravel app through it.

**Using Laravel Expose (by Beyond Code):**

```bash
sail composer require beyondcode/expose --dev
sail artisan expose:share my-app
# Gives you a public URL like https://my-app.sharedwithexpose.com
```

---

## 🌍 Deploying Laravel + Docker to a Real Server (Production)

### Overview of Deployment Options

| Method | Best For | Complexity | Cost |
|--------|---------|-----------|------|
| **Docker Compose on a VPS** | Most projects | Medium | Low (~$5/mo) |
| **Docker Swarm** | Multiple servers, high availability | High | Medium |
| **Kubernetes** | Large scale, enterprise | Very High | High |
| **Laravel Forge + Docker** | Managed, GUI-based | Low | ~$12/mo |
| **Railway / Render / Fly.io** | Quick deploys, no server management | Low | Free tier available |

---

### Method 1 — Docker Compose on a VPS (Most Common)

**Step 1 — Get a VPS** (DigitalOcean, Linode, Vultr, Hetzner — ~$5–6/month for 1GB RAM)

**Step 2 — Install Docker on the server:**

```bash
# SSH into your server
ssh root@your-server-ip

# Install Docker Engine
curl -fsSL https://get.docker.com | bash
sudo usermod -aG docker $USER
```

**Step 3 — Create a production `docker-compose.yml`** (different from your dev one — no Sail):

```yaml
version: "3.8"

services:

  app:
    image: your-dockerhub-username/my-laravel-app:latest
    restart: unless-stopped
    environment:
      APP_ENV: production
      APP_DEBUG: false
    volumes:
      - storage_data:/var/www/storage
    networks:
      - laravel_prod
    depends_on:
      - mysql
      - redis

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/prod.conf:/etc/nginx/conf.d/default.conf
      - certbot_data:/etc/letsencrypt
    networks:
      - laravel_prod
    depends_on:
      - app

  mysql:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_prod_data:/var/lib/mysql
    networks:
      - laravel_prod

  redis:
    image: redis:alpine
    restart: unless-stopped
    networks:
      - laravel_prod

networks:
  laravel_prod:
    driver: bridge

volumes:
  mysql_prod_data:
  storage_data:
  certbot_data:
```

**Step 4 — Build and push your production image:**

```bash
# On your development machine
docker build -t your-dockerhub-username/my-laravel-app:latest .
docker push your-dockerhub-username/my-laravel-app:latest
```

**Step 5 — On the server, pull and run:**

```bash
docker compose pull
docker compose up -d
docker compose exec app php artisan migrate --force
docker compose exec app php artisan config:cache
docker compose exec app php artisan route:cache
docker compose exec app php artisan storage:link
```

---

### Method 2 — Zero-Config Deploy with Railway (Easiest)

Railway detects your `Dockerfile` automatically and deploys with one command.

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login
railway login

# Deploy
railway up
```

Railway handles the server, SSL, domain, and scaling. Free tier available. No VPS to manage.

---

### Method 3 — Deploy with Fly.io

Fly.io runs your Docker container globally across multiple regions.

```bash
# Install flyctl
curl -L https://fly.io/install.sh | sh

# Login and create app
fly auth login
fly launch    # detects Dockerfile automatically

# Deploy
fly deploy
```

---

### Production Dockerfile (Optimised — No Dev Tools)

Your development Sail image includes tools you don't need in production (Node, npm dev dependencies, Xdebug). Use a lean production image instead:

```dockerfile
FROM php:8.4-fpm-alpine

# Install only production-required extensions
RUN apk add --no-cache \
    nginx \
    supervisor \
    libpng-dev \
    libxml2-dev \
    oniguruma-dev \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd opcache

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

# Copy app files
COPY . .

# Install PHP dependencies (production only — no dev packages)
RUN composer install --no-dev --optimize-autoloader --no-scripts

# Set permissions
RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache

# Cache Laravel config for speed
RUN php artisan config:cache \
 && php artisan route:cache \
 && php artisan view:cache

EXPOSE 9000
CMD ["php-fpm"]
```

> 📦 This production image is approximately **200–400 MB** — much smaller than the 2 GB Sail dev image, because it excludes Node.js, npm, Xdebug, and all dev tools.

---

### Setting Up HTTPS / SSL in Production

Use Certbot (Let's Encrypt — free SSL):

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx

# Get a certificate (replace with your actual domain)
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com

# Auto-renew is set up automatically — check with:
sudo certbot renew --dry-run
```

---

### Production `.env` Checklist

```env
APP_ENV=production
APP_DEBUG=false                      # ← CRITICAL: never true in production
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=mysql                        # ← service name, not localhost
DB_PORT=3306
DB_DATABASE=your_db
DB_USERNAME=your_user
DB_PASSWORD=strong_random_password   # ← use a strong password

CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

REDIS_HOST=redis
REDIS_PORT=6379
```

---

## 🧰 Pre-Flight Checklist Before Every `sail up`

Run through this list any time `sail up` fails or behaves unexpectedly:

```bash
# 1. Is Docker Desktop running? (Linux only)
systemctl --user status docker-desktop
# Expected: Active: active (running)
# Fix if not: systemctl --user start docker-desktop

# 2. Is the Docker context correct?
docker context ls
# Expected: desktop-linux marked with *
# Fix if not: docker context use desktop-linux

# 3. Can Docker respond at all?
docker info
# Expected: prints system info without errors
# If error: Docker is not running — start it first

# 4. Is anything using port 80 or 3306?
sudo lsof -i :80
sudo lsof -i :3306
# Fix if occupied: sudo service nginx stop / sudo service mysql stop

# 5. Does your .env file exist?
ls -la .env
# Fix if missing: cp .env.example .env && sail artisan key:generate

# 6. Does database/database.sqlite exist? (SQLite only)
ls database/database.sqlite
# Fix if missing: touch database/database.sqlite

# 7. Now start safely
./vendor/bin/sail up -d
```

---

## 🗺️ Quick Reference — All Commands in One Place

```bash
# ─── DOCKER DESKTOP ─────────────────────────────────────────────────────────
systemctl --user start docker-desktop      # Start Docker Desktop manually
systemctl --user enable docker-desktop     # Auto-start on login
systemctl --user stop docker-desktop       # Stop Docker Desktop
systemctl --user status docker-desktop     # Check if running

# ─── CONTEXT ────────────────────────────────────────────────────────────────
docker context ls                          # List contexts (* = active)
docker context use desktop-linux           # Use Docker Desktop
docker context use default                 # Use Docker Engine

# ─── BUILD & RESUME ─────────────────────────────────────────────────────────
./vendor/bin/sail up                       # Resume/start (resumes from cache)
./vendor/bin/sail up -d                    # Start in background
docker pull ubuntu:24.04                   # Pre-pull base image separately
docker pull laravelsail/php84-composer     # Pre-pull Sail image separately
docker pull mysql:8.0                      # Pre-pull MySQL separately

# ─── SAFE STOP ──────────────────────────────────────────────────────────────
sail down                                  # Stop containers (keeps images + DB)
sail down -v                               # Stop + delete DB volume (keeps images)

# ─── PORT CONFLICTS ─────────────────────────────────────────────────────────
sudo lsof -i :3306                         # Find what uses port 3306
sudo service mysql stop                    # Free port 3306
sudo service nginx stop                    # Free port 80

# ─── IMAGES — FIND & DELETE ─────────────────────────────────────────────────
docker images                              # List all images with sizes
docker rmi image-name:tag                  # Delete specific image by name
docker rmi abc123def456                    # Delete specific image by ID
docker rmi mysql:8.0 redis:alpine          # Delete multiple images at once
docker image prune                         # Delete dangling/unused images only
docker image prune -a                      # Delete ALL unused images
docker system df                           # Show total disk usage breakdown

# ─── SAVE & TRANSFER ────────────────────────────────────────────────────────
docker save -o images.tar image1 image2    # Export images to tar file
docker load -i images.tar                  # Import images from tar file

# ─── SQLITE SETUP ───────────────────────────────────────────────────────────
touch database/database.sqlite             # Create SQLite DB file

# ─── ARTISAN (inside container) ─────────────────────────────────────────────
sail artisan key:generate                  # Generate app key
sail artisan migrate                       # Run migrations
sail artisan migrate:fresh --seed          # Wipe + re-migrate + seed
sail artisan config:cache                  # Cache config (production)
sail artisan route:cache                   # Cache routes (production)
sail artisan optimize:clear                # Clear all caches (development)

# ─── HOT SHARING ────────────────────────────────────────────────────────────
hostname -I                                # Get your local IP
ngrok http 80                              # Share publicly via tunnel

# ─── PRODUCTION ─────────────────────────────────────────────────────────────
docker build -t username/app:latest .      # Build production image
docker push username/app:latest            # Push to Docker Hub
docker compose pull                        # Pull latest images on server
docker compose up -d                       # Start in production
```

---

> 💬 **Questions or issues?** Open an issue or reach out via [@johnboscocjt](https://github.com/johnboscocjt)
> ⭐ If this guide helped you, consider starring the repo!
