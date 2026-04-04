# 🐳 Docker + Laravel on Ubuntu — Complete Developer Guide

> **Author:** [@johnboscocjt](https://github.com/johnboscocjt)  
> **Platform:** Ubuntu 20.04 / 22.04 / 24.04  
> **Covers:** Docker Engine · Docker Desktop · Laravel Projects · Day-to-Day Workflow

---

## 📑 Table of Contents

0. [⚠️ IMPORTANT — Read This First: Bandwidth & Data Usage](#️-important--read-this-first-bandwidth--data-usage)
1. [Prerequisites](#prerequisites)
2. [Part 1 — Install Docker Engine (CLI)](#part-1--install-docker-engine-cli)
3. [Part 2 — Install Docker Desktop (GUI)](#part-2--install-docker-desktop-gui)
4. [Part 3 — Understanding Docker Contexts](#part-3--understanding-docker-contexts)
5. [Part 4 — Start a New Laravel Project with Docker](#part-4--start-a-new-laravel-project-with-docker)
6. [Part 5 — Daily Workflow: Start, Stop, Restart](#part-5--daily-workflow-start-stop-restart)
7. [Part 6 — Running Commands Inside Containers](#part-6--running-commands-inside-containers)
8. [Part 7 — Viewing Logs & Monitoring](#part-7--viewing-logs--monitoring)
9. [Part 8 — Using Docker Desktop GUI](#part-8--using-docker-desktop-gui)
10. [Part 9 — Common Errors & Fixes](#part-9--common-errors--fixes)
11. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
12. [Part 10 — Controlling Your Laravel Project via Docker Desktop](#part-10--controlling-your-laravel-project-via-docker-desktop)

---

## ⚠️ IMPORTANT — Read This First: Bandwidth & Data Usage

> 🚨 **This section is critical if you are on a limited internet plan, mobile data, or a slow connection.**
> Docker downloads large files — sometimes several gigabytes — during setup. **Skipping or misusing commands can cause duplicate downloads that waste your data.**
> Read this section fully before running any command in this guide.

---

### 📦 How Much Data Will Docker Download?

Below is a breakdown of every major download this guide triggers, with approximate sizes. These are **one-time downloads** — Docker caches everything locally. You only re-download if you delete images or rebuild from scratch.

#### 🔧 Installation Downloads (One-Time)

| What | Approximate Size | When It Downloads |
|------|-----------------|-------------------|
| Docker Engine (apt packages) | ~100–150 MB | During `apt install docker-ce ...` |
| Docker Desktop `.deb` installer | ~550–600 MB | When you run `wget docker-desktop-amd64.deb` |
| Docker Desktop internal VM setup | ~300–500 MB | On very first launch of Docker Desktop |
| **Installation total** | **~1.0–1.3 GB** | First time only |

> 💡 If you only need the terminal (no GUI), **skip Docker Desktop entirely** and save ~1 GB. Docker Engine alone is sufficient for all Laravel development.

---

#### 🐘 Laravel Project Downloads (One-Time per Project)

These are downloaded the first time you create or start a Laravel project. Once pulled, Docker reuses the cached images.

| Image / Package | Approximate Size | Used For |
|----------------|-----------------|----------|
| `laravelsail/php84-composer` | ~1.5–2.0 GB | Main Laravel app container (PHP 8.4 + Composer + Node) |
| `mysql:8.0` | ~550 MB | Database |
| `redis:alpine` | ~35 MB | Cache / queues |
| `nginx:alpine` | ~40 MB | Web server (custom setup only) |
| `mailpit` | ~50 MB | Local email testing |
| Composer vendor packages | ~50–150 MB | PHP dependencies (`composer install`) |
| Node modules (`npm install`) | ~200–500 MB | Frontend JS dependencies |
| **Project total (Sail + mysql + redis + mailpit)** | **~2.5–3.5 GB** | First `sail up` only |

> ⚠️ **The Laravel Sail image (`laravelsail/php84-composer`) is the biggest single download** — approximately 1.5 to 2 GB on its own. This is because it bundles PHP, Composer, Node.js, npm, and many system libraries all in one image.

---

#### 🔁 What Triggers Re-Downloads (Data You Can Avoid Wasting)

These actions cause Docker to download things again. **Avoid them unless necessary.**

| Action | Re-downloads? | Approximate Data Lost | How to Avoid |
|--------|--------------|----------------------|--------------|
| `docker system prune -a --volumes` | ✅ Yes — everything | 2–4 GB | Only run when you truly want a full reset |
| `docker image prune -a` | ✅ Yes — all images | 2–3 GB | Only prune images you no longer need |
| `sail build --no-cache` | ✅ Yes — rebuilds layers | 500 MB–1.5 GB | Only use when a build is genuinely broken |
| `sail down -v` | ❌ No images, but deletes DB data | 0 MB (data loss only) | Use `sail down` instead to keep your database |
| `sail down` | ❌ No | 0 MB | ✅ Safe — images kept, containers just stop |
| `sail up -d` (subsequent runs) | ❌ No | 0 MB | ✅ Safe — reuses cached images every time |
| Failing `laravel.build` without `?php=84` | ⚠️ Partial waste | ~200–500 MB wasted | Always specify `?php=84` in the URL |

---

### 🧠 The Most Important Rules for Limited Data Users

#### Rule 1 — Always specify the PHP version when creating a project

```bash
# ❌ WRONG — will try PHP 8.5, fail, and waste ~200–500 MB on a broken pull
curl -s "https://laravel.build/my-app" | bash

# ✅ CORRECT — pulls the right image on the first attempt, no wasted data
curl -s "https://laravel.build/my-app?php=84" | bash
```

**What happens if you don't specify?** Laravel defaults to the latest PHP version (currently 8.5). Docker will attempt to pull `laravelsail/php85-composer`. If that image doesn't exist yet on Docker Hub, Docker downloads partial manifest and layer data (~200–500 MB depending on how far it gets), then fails with an error — and you've lost that data for nothing. You then have to run the command again with the correct version, downloading even more.

---

#### Rule 2 — Never use `--no-cache` or `prune` unless something is genuinely broken

```bash
# ❌ Wastes 500 MB–1.5 GB — forces a full image rebuild from scratch
sail build --no-cache

# ❌ Deletes ALL images — you re-download everything next time (2–4 GB)
docker system prune -a --volumes

# ✅ Safe restart — reuses all cached images, downloads nothing
sail down && sail up -d
```

---

#### Rule 3 — Use `sail down` NOT `sail down -v` for daily stops

```bash
# ❌ This deletes your database volume (data loss) — does NOT re-download images
#    but your database is gone and you must run migrations again
sail down -v

# ✅ This safely stops all containers and keeps your database intact
sail down
```

> `-v` stands for **volumes** — it destroys your MySQL data. Use it only when you intentionally want a clean database (e.g., switching branches with schema conflicts). It does not cause image re-downloads, but it means your next `sail artisan migrate` starts with an empty database.

---

#### Rule 4 — Once images are pulled, starting/stopping costs zero data

After the initial setup, your daily workflow costs **zero additional downloads**:

```bash
sail up -d             # 0 MB — starts from cached images
sail down              # 0 MB — stops containers, keeps everything
sail restart           # 0 MB — restarts from cache
sail artisan migrate   # 0 MB — runs inside existing container
```

Docker does not re-download images every time you start your project. Images are stored locally on your disk and reused indefinitely.

---

#### Rule 5 — Skip Docker Desktop if you are on a limited data plan

Docker Desktop adds **~1.0–1.1 GB** of downloads just for the installer and its internal VM. If you are comfortable with the terminal, **Docker Engine alone is everything you need.** All commands in this guide work without Docker Desktop.

Only install Docker Desktop if you specifically need the visual GUI for logs, stats, or container management.

---

### 📊 Complete Data Budget Summary

| Scenario | Total Download | Notes |
|----------|---------------|-------|
| Docker Engine only (no Desktop) | ~100–150 MB | Minimal setup, terminal only |
| Docker Engine + Docker Desktop | ~1.0–1.3 GB | Adds visual GUI |
| First Laravel project (Sail + mysql + redis) | ~2.5–3.0 GB | One-time image pull |
| First Laravel project (custom nginx setup) | ~700 MB–1.0 GB | Smaller images, no Sail |
| Daily use after setup | **0 MB** | Everything cached locally |
| Full reset (`system prune -a`) | Re-downloads 2–4 GB | Avoid unless truly needed |
| Failed `laravel.build` (no PHP version) | ~200–500 MB wasted | Always use `?php=84` |

> 🏁 **Bottom line:** Expect to spend **3–5 GB total** on the very first setup. After that, your daily usage is **0 MB** of additional downloads. Protect that initial investment — don't prune images unless you have a specific reason.

---

### ⚖️ Bandwidth Comparison: Traditional Laravel vs Docker

> This is one of the most important decisions you'll make. Here is an honest, side-by-side breakdown of what each approach actually downloads so you can choose what's right for your internet situation.

---

#### 🛤️ Method 1 — Traditional Laravel (No Docker)

This is the classic way: install PHP, Composer, MySQL, and Node directly on your machine using your system's package manager.

**What gets installed and how much data it costs:**

| Step | Command | Approximate Size | What It Installs |
|------|---------|-----------------|------------------|
| 1. Install PHP + extensions | `sudo apt install php8.4 php8.4-mbstring php8.4-xml php8.4-curl php8.4-mysql ...` | ~50–80 MB | PHP runtime + required extensions |
| 2. Install Composer | `curl ... \| php -- --install-dir=...` | ~2–5 MB | PHP package manager |
| 3. Create Laravel project | `composer create-project laravel/laravel my-app` | ~50–80 MB | Laravel framework + all PHP dependencies |
| 4. Install MySQL | `sudo apt install mysql-server` | ~200–250 MB | MySQL database server |
| 5. Install Node.js + npm | `sudo apt install nodejs npm` | ~70–100 MB | JavaScript runtime |
| 6. Install frontend deps | `npm install` (inside project) | ~150–300 MB | Vite, Tailwind, etc. |
| **Total (first time)** | | **~520–815 MB** | Full working Laravel environment |

**Subsequent projects on the same machine:**

| Step | Cost | Reason |
|------|------|--------|
| PHP, MySQL, Node already installed | 0 MB | System-wide, reused by every project |
| `composer create-project laravel/laravel my-app2` | ~30–50 MB | Only downloads packages not already in Composer cache |
| `npm install` | ~50–150 MB | Some packages cached by npm, some new |
| **Total for 2nd project** | **~80–200 MB** | Much cheaper after first setup |

---

#### 🐳 Method 2 — Laravel with Docker (Sail)

Docker packages each project's entire environment into isolated containers. Every service (PHP, MySQL, Redis, Node) runs inside its own image.

**What gets downloaded and how much data it costs:**

| Step | Command | Approximate Size | What It Downloads |
|------|---------|-----------------|------------------|
| 1. Install Docker Engine | `apt install docker-ce ...` | ~100–150 MB | Docker daemon + CLI tools |
| 2. (Optional) Docker Desktop | `apt install docker-desktop-amd64.deb` | ~850 MB–1.1 GB | GUI + internal VM |
| 3. Pull Sail image | Auto on first `sail up` | ~1.5–2.0 GB | PHP 8.4 + Composer + Node + npm + extensions |
| 4. Pull MySQL image | Auto on first `sail up` | ~550 MB | Full MySQL 8.0 server |
| 5. Pull Redis image | Auto on first `sail up` | ~35 MB | Redis server |
| 6. Pull Mailpit image | Auto on first `sail up` | ~50 MB | Email testing server |
| 7. Composer install | `sail composer install` | ~50–80 MB | PHP packages (runs inside container) |
| 8. npm install | `sail npm install` | ~150–300 MB | Frontend packages (runs inside container) |
| **Total (Engine + Sail + all services)** | | **~2.6–3.4 GB** | Full Docker Laravel stack |
| **Total (Engine only, no Desktop)** | | **~2.1–2.8 GB** | Same stack, no GUI |

**Subsequent projects on the same machine:**

| Step | Cost | Reason |
|------|------|--------|
| Docker Engine already installed | 0 MB | Installed once, used forever |
| Sail image already pulled | 0 MB | Same PHP version = same cached image |
| MySQL image already pulled | 0 MB | `mysql:8.0` cached from first project |
| Redis / Mailpit already pulled | 0 MB | Cached |
| `composer install` | ~30–50 MB | Only new/changed packages |
| `npm install` | ~50–150 MB | Some packages cached |
| **Total for 2nd project** | **~80–200 MB** | Same as traditional — images are reused |

> 💡 After the first Docker project, subsequent projects are just as cheap as traditional Laravel — because all the heavy images are already cached on your machine.

---

#### 📊 Head-to-Head Comparison Table

| | Traditional Laravel | Docker (Sail, Engine only) | Docker (Sail + Desktop) |
|--|--------------------|-----------------------------|------------------------|
| **First project download** | ~520–815 MB | ~2.1–2.8 GB | ~2.6–3.4 GB |
| **Second project download** | ~80–200 MB | ~80–200 MB | ~80–200 MB |
| **Daily start/stop cost** | 0 MB | 0 MB | 0 MB |
| **PHP version conflicts** | ⚠️ Risk (system-wide PHP) | ✅ None (isolated per project) |✅ None |
| **MySQL conflicts** | ⚠️ Risk (one system MySQL) | ✅ None (each project isolated) | ✅ None |
| **Works on teammate's machine** | ⚠️ "Works on my machine" risk | ✅ Identical environment | ✅ Identical environment |
| **Wipe and reset a project** | ❌ Manual and messy | ✅ `sail down -v && sail up -d` | ✅ Same |
| **Good for limited internet** | ✅ Yes — much lighter first setup | ⚠️ Only after initial large download | ❌ High first cost |
| **Disk space used** | ~800 MB–1.5 GB | ~4–6 GB (images + volumes) | ~5–7 GB |

---

#### 🤔 Which Should You Choose?

**Choose Traditional Laravel if:**
- You are on a severely limited data plan and this is your first time setting up
- You only run one Laravel version at a time
- You are a solo developer and don't need to match teammates' environments
- You're comfortable managing PHP, MySQL, and Node directly on your system

```bash
# Traditional setup (lighter on data)
sudo apt install php8.4 php8.4-cli php8.4-mbstring php8.4-xml php8.4-curl php8.4-mysql php8.4-zip
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer create-project laravel/laravel my-app
cd my-app && php artisan serve
# Open http://localhost:8000
```

**Total first-time cost: ~520–815 MB**

---

**Choose Docker (Sail) if:**
- You have a decent data allowance for the initial setup (3–4 GB)
- You work on multiple Laravel projects that may need different PHP versions
- You work in a team and need everyone's environment to match exactly
- You want a clean, disposable environment with no risk of polluting your system

```bash
# Docker setup (heavier upfront, but better isolation)
curl -s "https://laravel.build/my-app?php=84" | bash
cd my-app && sail up -d
# Open http://localhost
```

**Total first-time cost: ~2.1–3.4 GB**

---

#### 💾 `laravel new` vs `composer create-project` — Are They Different?

Many developers wonder if `laravel new` and `composer create-project laravel/laravel` use different amounts of data. Here's the truth:

| Method | Extra Download | Notes |
|--------|---------------|-------|
| `composer create-project laravel/laravel my-app` | 0 MB extra | Downloads Laravel directly via Composer — no extra tools |
| `laravel new my-app` (Laravel Installer) | ~2–5 MB extra (one-time) | Requires installing the Laravel Installer globally first: `composer global require laravel/installer` |
| **The actual Laravel project download** | **~50–80 MB** | **Same for both methods** — same packages, same size |

```bash
# Method A — No extra tools needed (~50–80 MB total)
composer create-project laravel/laravel my-app

# Method B — Install Laravel Installer once, then use it forever (~2 MB extra, one-time)
composer global require laravel/installer   # one-time, ~2–5 MB
laravel new my-app                          # ~50–80 MB each time (same as Method A)
```

> ✅ **Both methods produce the exact same Laravel project** with the same file size and the same dependencies. `laravel new` is just a convenience wrapper — it does not download anything extra for the project itself. The only difference is the one-time install of the Laravel Installer tool (~2–5 MB).

---

#### 📉 Visual Summary: Data Cost Over 3 Projects

```
Traditional Laravel
────────────────────────────────────────────────────────────
Project 1:  ████████████████████████████████  ~700 MB
Project 2:  ████  ~120 MB
Project 3:  ████  ~120 MB
Total:      ████████████████████████████████████████  ~940 MB

Docker (Sail) — Engine Only
────────────────────────────────────────────────────────────
Project 1:  ████████████████████████████████████████████████████████████████  ~2.5 GB
Project 2:  ████  ~120 MB
Project 3:  ████  ~120 MB
Total:      ████████████████████████████████████████████████████████████████████  ~2.74 GB

Docker (Sail) — With Docker Desktop
────────────────────────────────────────────────────────────
Project 1:  ██████████████████████████████████████████████████████████████████████████  ~3.2 GB
Project 2:  ████  ~120 MB
Project 3:  ████  ~120 MB
Total:      ████████████████████████████████████████████████████████████████████████████  ~3.44 GB
```

> 🔑 **Key takeaway:** Traditional Laravel is significantly cheaper for your first project. By the third project, the gap narrows because Docker reuses all its cached images. If you plan to build many Laravel projects over time, Docker's upfront cost pays for itself in consistency and isolation.

---

### 🔄 Image Reuse — How It Works Across Projects

> This is the most misunderstood part of Docker for new users. Understanding this will save you gigabytes of unnecessary downloads and help you manage your disk space wisely.

---

#### Do Docker Images Get Reused When I Create a New Project?

**Yes — absolutely.** Docker images are stored globally on your machine inside Docker's own storage area (`/var/lib/docker/`), completely separate from your project folders. When you start a second Laravel project, Docker checks its local image cache first. If the image is already there, it uses it instantly — no download.

```
First project (my-app)          Second project (my-blog)
─────────────────────           ─────────────────────────
sail up -d                      sail up -d
  └─ laravelsail/php84 ──┐        └─ laravelsail/php84 ──┘ ← REUSED, 0 MB
  └─ mysql:8.0 ──────────┤        └─ mysql:8.0 ──────────┘ ← REUSED, 0 MB
  └─ redis:alpine ───────┘        └─ redis:alpine ─────────┘ ← REUSED, 0 MB
                ↓                               ↓
        Downloads ~2.5 GB             Downloads ~0 MB (images cached)
                                       Only ~80–200 MB for new
                                       Composer + npm packages
```

**The rule:** Same image name + same tag = always reused. As long as project 2 uses the same PHP version (`php=84`) and same services (`mysql:8.0`, `redis:alpine`), Docker pulls nothing for the environment.

---

#### Does Deleting My Project Folder Delete the Docker Images?

**No — never.** Deleting your project folder (`rm -rf my-app`) only removes your code files. The Docker images stay on disk inside Docker's internal storage and remain available for every future project.

Here is exactly what each action deletes and what it leaves alone:

| Action | Your Code Files | Docker Images (the GBs) | Database Data | Download Again? |
|--------|----------------|------------------------|---------------|----------------|
| `rm -rf my-app` | ✅ Gone | ❌ Safe | ❌ Safe | ❌ No |
| `sail down` | ❌ Safe | ❌ Safe | ❌ Safe | ❌ No |
| `sail down -v` | ❌ Safe | ❌ Safe | ✅ Gone | ❌ No |
| `docker rm <container>` | ❌ Safe | ❌ Safe | ❌ Safe | ❌ No |
| `docker rmi <image>` | ❌ Safe | ✅ That image only | ❌ Safe | ✅ Yes, that image only |
| `docker image prune -a` | ❌ Safe | ✅ ALL images gone | ❌ Safe | ✅ Yes, everything |
| `docker system prune -a --volumes` | ❌ Safe | ✅ ALL images gone | ✅ Gone | ✅ Yes, full re-download |

> 🚨 **Critical:** `docker system prune -a --volumes` is the **only command that forces a full re-download** of everything (2–4 GB). Never run it casually. It is a last-resort reset tool, not a cleanup routine.

---

### 🔍 Finding & Deleting Specific Docker Images

#### Via Terminal (CLI)

**Step 1 — List all images currently on your machine:**

```bash
docker images
```

Example output:

```
REPOSITORY                    TAG       IMAGE ID       CREATED        SIZE
laravelsail/php84-composer    latest    a1b2c3d4e5f6   2 weeks ago    2.01GB
mysql                         8.0       b2c3d4e5f6a7   3 weeks ago    587MB
redis                         alpine    c3d4e5f6a7b8   1 month ago    41MB
nginx                         alpine    d4e5f6a7b8c9   1 month ago    43MB
mailpit/mailpit               latest    e5f6a7b8c9d0   5 weeks ago    52MB
```

**Step 2 — Delete a specific image by name:**

```bash
# Delete by repository name and tag
docker rmi laravelsail/php84-composer:latest

# Delete by IMAGE ID (first few characters are enough)
docker rmi a1b2c3d4e5f6

# Delete multiple images at once
docker rmi mysql:8.0 redis:alpine nginx:alpine
```

> ⚠️ You cannot delete an image while a container is using it. Stop and remove the container first:
> ```bash
> sail down          # stops and removes containers
> docker rmi <image> # now safe to delete
> ```

**Step 3 — Delete all unused images (images not attached to any running container):**

```bash
# Remove only dangling images (untagged, intermediate build layers — usually safe)
docker image prune

# Remove ALL images not currently used by a running container
docker image prune -a
```

**Step 4 — Check how much disk space Docker is using before and after:**

```bash
# Shows disk usage broken down by images, containers, volumes, build cache
docker system df
```

Example output:

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         2         2.71GB    1.5GB (55%)
Containers      3         2         1.2MB     0B (0%)
Local Volumes   2         1         450MB     0MB (0%)
Build Cache     12        0         380MB     380MB
```

**Step 5 — Nuclear option (wipes everything — use with caution):**

```bash
# Removes all stopped containers, all unused images, all unused volumes, build cache
docker system prune -a --volumes
```

> 🚨 After this command, your next `sail up` will re-download 2–4 GB. Only use this if you want to completely reset Docker on your machine.

---

#### Via Docker Desktop (GUI)

**Finding and inspecting images:**

1. Open Docker Desktop
2. Click the **Images** tab in the left sidebar
3. You will see a list of all images with their **name**, **tag**, **size**, and **date created**
4. Use the **search bar** at the top to filter by name (e.g., type `mysql` to find only MySQL images)
5. Click any image row to see its full details — layers, tags, and which containers are using it

**Deleting a specific image:**

1. **Images** tab → find the image you want to remove
2. Hover over the row — a 🗑️ **Delete** button appears on the right
3. Click it → confirm the deletion
4. The image is immediately removed from your disk

**Deleting multiple images at once:**

1. **Images** tab → tick the **checkboxes** on the left of each image you want to remove
2. A toolbar appears at the top with a **Delete** button
3. Click **Delete** → confirm — all selected images are removed together

**Deleting all unused images (bulk clean):**

1. **Images** tab → click **Clean up** button (top right area)
2. Docker Desktop shows you all unused images with their sizes
3. Select the ones to remove or click **Select all**
4. Click **Remove** — reclaims disk space without touching images in use

**Checking total disk usage in Docker Desktop:**

1. Click **Settings (⚙️)** → **Resources** → **Disk usage**
2. You can see a visual breakdown of how much disk Docker is consuming
3. There is a **Purge data** button here as a last resort — equivalent to `docker system prune -a --volumes`

---

### 🏆 The Real Advantage: What Happens After Your First Project

> The first Docker project is painful on data. Here is what you gain from project 2 onwards — and why most professional developers consider it worth it.

---

#### 📉 Data Cost After the First Project

Once your images are cached, every new Laravel project you create costs approximately:

| What downloads | Approximate Size | Why |
|---------------|-----------------|-----|
| Laravel framework files | ~50–80 MB | `composer create-project` or `laravel.build` |
| Your project's Composer packages | ~30–80 MB | Unique dependencies of your new project |
| Your project's npm packages | ~50–200 MB | Vite, Tailwind, Alpine.js, etc. |
| Docker images (PHP, MySQL, Redis) | **0 MB** | Already cached from project 1 |
| **Total per new project** | **~130–360 MB** | Same or less than traditional Laravel |

This is cheaper than or equal to starting a traditional Laravel project for the second time.

---

#### ✅ Advantages You Gain From Project 2 Onwards

**1. Instant environment — zero setup time**

On a traditional setup, every new machine or teammate needs to install PHP, MySQL, Node, configure versions, handle conflicts. With Docker, your second project spins up in seconds because the environment is already there:

```bash
# Everything ready in under 30 seconds — no installs, no config
cd my-second-project
sail up -d
# Done. Open http://localhost
```

**2. Run multiple projects with different PHP versions simultaneously — no conflicts**

Traditional Laravel forces one system-wide PHP version. With Docker, project 1 can run PHP 8.2 while project 2 runs PHP 8.4 — both at the same time, zero conflicts:

```bash
# Terminal 1 — project on PHP 8.2 (port 80)
cd my-old-project && sail up -d

# Terminal 2 — project on PHP 8.4 (port 8080)
cd my-new-project && sail up -d
```

**3. Completely isolated databases — no cross-project pollution**

Each project gets its own MySQL volume. Data from `my-app` never bleeds into `my-blog`. Drop all tables in one project — the other is untouched.

**4. Destroy and rebuild any project's environment in one command**

Had a corrupted database or a broken package? On traditional Laravel this can take hours to fix. With Docker:

```bash
sail down -v          # wipe containers + database
sail up -d            # fresh environment in seconds
sail artisan migrate --seed  # data restored
```

**5. New team members are productive in minutes, not hours**

A teammate clones your project and runs two commands — that's it. No "install PHP 8.4", no "configure MySQL root password", no debugging why it works on your machine but not theirs:

```bash
git clone https://github.com/yourteam/my-app
cd my-app && sail up -d
# Running. Identical to your environment.
```

If they've worked on any other Docker Laravel project before, they already have the images cached — their setup downloads **0 MB** of Docker images.

**6. Safe cleanup — delete a project without touching your system**

On traditional Laravel, uninstalling a project means hunting down config files, database users, PHP extensions. With Docker, deletion is surgical:

```bash
sail down -v          # removes containers + database volume
rm -rf my-app         # removes code
# System is 100% clean. Nothing left behind.
```

The Docker images stay cached for your next project — you only deleted what belonged to this project.

---

#### 📊 Cost vs Benefit Summary Over Time

| | Project 1 | Project 2 | Project 3+ | Cumulative Advantage |
|--|-----------|-----------|------------|---------------------|
| **Traditional Laravel** | ~700 MB | ~150 MB | ~150 MB each | Simple but version conflicts grow over time |
| **Docker (Sail)** | ~2.5 GB | ~150 MB | ~150 MB each | Isolation, reproducibility, zero conflicts |
| **Data gap** | Docker costs ~1.8 GB more | **Equal** | **Equal** | Docker's upfront cost is a one-time investment |
| **Time gap** | Docker takes longer first time | Docker is **faster** | Docker is **faster** | No environment setup from project 2 onwards |

> 🎯 **The honest verdict:** If you only ever build one Laravel project, traditional is lighter. If you build two or more — especially with a team — Docker pays back its upfront data cost in time saved, conflicts avoided, and environments that just work.

---

Before starting, make sure your Ubuntu system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

Check your Ubuntu version:

```bash
cat /etc/os-release
```

You should be on **Ubuntu 20.04, 22.04, or 24.04** (Desktop edition — not Server — if you want Docker Desktop).

---

## Part 1 — Install Docker Engine (CLI)

> Docker Engine is the core Docker daemon that runs in the background. Always install this first — Docker Desktop depends on it too.

### Step 1.1 — Remove old/conflicting Docker packages

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y $pkg
done
```

### Step 1.2 — Install required dependencies

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### Step 1.3 — Add Docker's official GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Step 1.4 — Add Docker's repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 1.5 — Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Step 1.6 — Add your user to the docker group (avoid using `sudo` every time)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> ⚠️ Log out and back in (or reboot) for this to fully take effect.

### Step 1.7 — Verify Docker Engine is working

```bash
docker --version
docker compose version
docker run hello-world
```

✅ You should see `Hello from Docker!` — Docker Engine is installed and working.

### Step 1.8 — Enable Docker to start on boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

## Part 2 — Install Docker Desktop (GUI)

> Docker Desktop gives you a visual dashboard to manage containers, images, and volumes. It runs its **own internal VM** separate from Docker Engine.

### Step 2.1 — Install GNOME Terminal (required dependency)

```bash
sudo apt update
sudo apt install -y gnome-terminal
```

### Step 2.2 — Download Docker Desktop `.deb` package

```bash
wget https://desktop.docker.com/linux/main/amd64/docker-desktop-amd64.deb
```

> 💡 This downloads the latest stable release directly from Docker.

### Step 2.3 — Install Docker Desktop

```bash
sudo apt install ./docker-desktop-amd64.deb
```

If you see dependency errors, fix them with:

```bash
sudo apt --fix-broken install
```

### Step 2.4 — Launch Docker Desktop

**Option A — From your Applications Menu (recommended):**  
Search for **"Docker Desktop"** in your app launcher and click it.

**Option B — From Terminal:**

```bash
systemctl --user start docker-desktop
```

> The first launch may take a few minutes as it sets up the internal VM. You'll see the 🐳 Docker whale icon in your system tray when it's ready.

### Step 2.5 — Verify Docker Desktop is running

```bash
docker --version
docker context ls
```

You should see `desktop-linux` listed as one of the contexts.

---

## Part 3 — Understanding Docker Contexts

> **This is the most important concept when using both Docker Engine and Docker Desktop on the same machine.**

Docker Engine and Docker Desktop use **separate daemons** (separate backends). Containers you create in one context **won't appear** in the other.

### View all contexts

```bash
docker context ls
```

Output example:

```
NAME              DESCRIPTION                               DOCKER ENDPOINT
default           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux *   Docker Desktop                            ...
```

The `*` shows the currently active context.

### Switch to Docker Desktop context

```bash
docker context use desktop-linux
```

### Switch back to Docker Engine (native)

```bash
docker context use default
```

### 💡 Recommendation

Pick **one context** for your Laravel projects and stick with it. Most developers use:
- `default` → if you work mostly from the terminal
- `desktop-linux` → if you prefer the visual Docker Desktop dashboard

---

## Part 4 — Start a New Laravel Project with Docker

There are two common approaches. **Option A** (Laravel Sail) is the easiest for beginners.

---

### Option A — Laravel Sail (Recommended for Beginners)

Laravel Sail is Laravel's official Docker development environment.

#### Step 4A.1 — Create a new Laravel project using Sail

Start with the basic command — no extra options needed:

```bash
curl -s "https://laravel.build/my-app" | bash
```

Replace `my-app` with your project name. You can run this **from any folder** — it will create the project as a subfolder wherever your terminal currently is.

> #### ⚠️ If you see this error:
> ```
> docker: Error response from daemon: pull access denied for laravelsail/php85-composer,
> repository does not exist or may require 'docker login'
> ```
> **What happened:** `laravel.build` automatically picks the latest PHP version (e.g. PHP 8.5). If a Sail Docker image for that version hasn't been published yet — because it's too new — Docker can't pull it and the whole setup fails. Your project folder either won't be created or will be left empty and broken.
>
> **The fix:** Specify a PHP version that actually has a published image:
>
> ```bash
> # Remove the broken folder first (if it was partially created)
> rm -rf my-app
>
> # Re-run with a specific PHP version
> curl -s "https://laravel.build/my-app?php=84" | bash
> ```
>
> **Available PHP versions:**
>
> | PHP Version | URL Parameter |
> |-------------|---------------|
> | PHP 8.4 ✅ (recommended) | `?php=84` |
> | PHP 8.3 | `?php=83` |
> | PHP 8.2 | `?php=82` |

To include specific services (e.g., MySQL, Redis, Mailpit):

```bash
curl -s "https://laravel.build/my-app?with=mysql,redis,mailpit&php=84" | bash
```

#### Step 4A.2 — Enter the project directory

```bash
cd my-app
```

#### Step 4A.3 — Start the project

```bash
./vendor/bin/sail up
```

Or run in the background (detached mode):

```bash
./vendor/bin/sail up -d
```

#### Step 4A.4 — Create a shell alias for convenience

Instead of typing `./vendor/bin/sail` every time, add this alias to your shell:

```bash
echo "alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'" >> ~/.bashrc
source ~/.bashrc
```

Now you can just type `sail` instead of `./vendor/bin/sail`.

#### Step 4A.5 — Open your app in the browser

```
http://localhost
```

---

### Option B — Custom `docker-compose.yml` (Full Control)

Use this approach when you need custom configuration or are integrating Docker into an existing project.

#### Step 4B.1 — Create your project directory

```bash
mkdir my-laravel-app && cd my-laravel-app
```

#### Step 4B.2 — Create a `docker-compose.yml` file

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
version: "3.8"

services:

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: laravel_app
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - .:/var/www
    networks:
      - laravel

  nginx:
    image: nginx:alpine
    container_name: laravel_nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - .:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    networks:
      - laravel

  mysql:
    image: mysql:8.0
    container_name: laravel_mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: root
      MYSQL_PASSWORD: secret
      MYSQL_USER: laravel
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - laravel

networks:
  laravel:
    driver: bridge

volumes:
  mysql_data:
```

#### Step 4B.3 — Create a `Dockerfile`

```bash
nano Dockerfile
```

```dockerfile
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    nodejs \
    npm

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

COPY . .

RUN composer install --no-scripts --no-autoloader

RUN composer dump-autoload

EXPOSE 9000
CMD ["php-fpm"]
```

#### Step 4B.4 — Create the Nginx config

```bash
mkdir -p docker/nginx
nano docker/nginx/default.conf
```

```nginx
server {
    listen 80;
    index index.php index.html;
    root /var/www/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### Step 4B.5 — Create a new Laravel app inside the container

```bash
docker compose run --rm app composer create-project laravel/laravel .
```

#### Step 4B.6 — Build and start everything

```bash
docker compose up -d --build
```

#### Step 4B.7 — Open your app

```
http://localhost:8080
```

---

## Part 5 — Daily Workflow: Start, Stop, Restart

### Using Laravel Sail

| Action | Command |
|--------|---------|
| Start containers | `sail up -d` |
| Stop containers | `sail down` |
| Restart containers | `sail restart` |
| Stop & remove volumes | `sail down -v` |
| Rebuild containers | `sail build --no-cache` |
| View running containers | `sail ps` |

### Using Docker Compose (custom setup)

| Action | Command |
|--------|---------|
| Start containers | `docker compose up -d` |
| Stop containers | `docker compose down` |
| Restart containers | `docker compose restart` |
| Stop & remove volumes | `docker compose down -v` |
| Rebuild containers | `docker compose up -d --build` |
| View running containers | `docker compose ps` |

### General Docker Commands

```bash
# List all running containers
docker ps

# List all containers (including stopped ones)
docker ps -a

# List all images
docker images

# Stop a specific container
docker stop <container_name_or_id>

# Start a stopped container
docker start <container_name_or_id>

# Remove a stopped container
docker rm <container_name_or_id>

# Remove an image
docker rmi <image_name_or_id>

# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune -a

# Remove everything (containers, images, volumes, networks) — USE WITH CAUTION
docker system prune -a --volumes
```

---

## Part 6 — Running Commands Inside Containers

### Using Laravel Sail

```bash
# Run Artisan commands
sail artisan migrate
sail artisan make:controller UserController
sail artisan make:model Post -m
sail artisan cache:clear
sail artisan config:clear
sail artisan route:list

# Composer
sail composer install
sail composer require laravel/sanctum
sail composer update

# NPM / Node
sail npm install
sail npm run dev
sail npm run build

# Open a bash shell inside the app container
sail bash

# Run PHP directly
sail php --version
sail php artisan tinker

# Run tests
sail test
sail artisan test
```

### Using Docker Compose (custom setup)

```bash
# Enter the app container shell
docker compose exec app bash

# Or for a one-off command without entering the shell:
docker compose exec app php artisan migrate
docker compose exec app php artisan make:controller UserController
docker compose exec app composer install
docker compose exec app php artisan cache:clear

# Run a command in the MySQL container
docker compose exec mysql mysql -u laravel -psecret laravel
```

### Using `docker exec` directly

```bash
# Enter a running container by name
docker exec -it laravel_app bash

# Run a single command
docker exec laravel_app php artisan migrate

# Run as a specific user
docker exec -it --user www-data laravel_app bash
```

---

## Part 7 — Viewing Logs & Monitoring

### View container logs

```bash
# View logs for all services (Sail)
sail logs

# Follow logs in real-time (Sail)
sail logs -f

# View logs for a specific service (Sail)
sail logs nginx
sail logs mysql

# Docker Compose — all services
docker compose logs

# Docker Compose — follow all logs
docker compose logs -f

# Docker Compose — specific service
docker compose logs -f app
docker compose logs -f nginx
docker compose logs -f mysql

# Native Docker — view logs by container name
docker logs laravel_app
docker logs -f laravel_app          # follow (live stream)
docker logs --tail=100 laravel_app  # last 100 lines only
```

### Laravel application logs

```bash
# View Laravel logs from inside the project
tail -f storage/logs/laravel.log

# Or from inside the container
docker compose exec app tail -f storage/logs/laravel.log
```

### Monitor resource usage

```bash
# Live CPU/memory/network stats for all running containers
docker stats

# Stats for a specific container
docker stats laravel_app

# View detailed container info
docker inspect laravel_app
```

### Check container status

```bash
# Running containers only
docker ps

# All containers (running + stopped)
docker ps -a

# With disk size info
docker ps --size
```

---

## Part 8 — Using Docker Desktop GUI

Docker Desktop provides a visual interface for everything you've been doing in the terminal.

### Starting Docker Desktop

- Search **"Docker Desktop"** in your application launcher and click it, OR
- Run from terminal: `systemctl --user start docker-desktop`

Wait for the 🐳 whale icon to appear in your system tray — that means it's running.

### Containers Tab

- See **all running and stopped containers** at a glance
- Click a container to see its **logs in real-time**
- Use the ▶️ / ⏹ / 🔄 buttons to **start, stop, or restart** a container
- Click the **Terminal** icon `>_` to open a shell directly inside the container
- Click the **Stats** tab to see **CPU and memory usage**

### Images Tab

- View all downloaded Docker images
- See image **size and creation date**
- Delete images you no longer need with the 🗑️ button
- **Pull** new images by clicking "Pull"

### Volumes Tab

- View all named volumes (like your MySQL database data)
- Inspect volume contents
- Delete volumes you no longer need

### Dev Environments Tab

- Create shareable dev environments (useful for teams)

### Settings (Gear Icon ⚙️)

| Setting | What it does |
|---------|-------------|
| Resources → CPU | Limit how many CPU cores Docker Desktop VM uses |
| Resources → Memory | Set max RAM (default 2GB — increase for larger apps) |
| Resources → Disk | Set how much disk space Docker can use |
| General → Start on login | Auto-start Docker Desktop when you log in |
| Docker Engine | Edit the raw Docker daemon JSON config |

### Switching contexts in Docker Desktop

Docker Desktop defaults to the `desktop-linux` context. To make sure you're managing the right containers, always match your terminal context:

```bash
# To align terminal with Docker Desktop
docker context use desktop-linux

# To use native Docker Engine (containers won't show in Desktop)
docker context use default
```

---

## Part 9 — Common Errors & Fixes

### ❌ `pull access denied for laravelsail/php85-composer` (or any php8x)

**Cause:** Running `curl -s "https://laravel.build/my-app" | bash` without specifying a PHP version causes Laravel to default to the latest PHP (e.g. 8.5), which may not have a Sail Docker image published yet. The project folder gets created but is empty or broken.

**Fix:** Always pass a valid `?php=` version in the URL:

```bash
# ✅ Correct — specify PHP 8.4 explicitly
curl -s "https://laravel.build/my-app?php=84" | bash

# ✅ With services too
curl -s "https://laravel.build/my-app?with=mysql,redis,mailpit&php=84" | bash
```

If the folder was partially created from a failed attempt, remove it first:

```bash
rm -rf my-app
curl -s "https://laravel.build/my-app?php=84" | bash
```

**Valid PHP options:** `php=82`, `php=83`, `php=84`

---

### ❌ `docker: permission denied`

**Cause:** Your user isn't in the `docker` group.

```bash
sudo usermod -aG docker $USER
newgrp docker
# Then log out and back in
```

---

### ❌ `Port is already allocated` or `bind: address already in use`

**Cause:** Something else is using the port (e.g., port 80 or 3306).

```bash
# Find what's using port 80
sudo lsof -i :80
sudo lsof -i :3306

# Kill the process using the port
sudo kill -9 <PID>

# Or change the port in docker-compose.yml
ports:
  - "8081:80"   # change 8080 to something else
```

---

### ❌ `Cannot connect to the Docker daemon`

**Cause:** Docker isn't running.

```bash
# Start Docker Engine
sudo systemctl start docker

# Start Docker Desktop
systemctl --user start docker-desktop
```

---

### ❌ `Sail: command not found`

**Cause:** Shell alias not set up.

```bash
echo "alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'" >> ~/.bashrc
source ~/.bashrc
```

---

### ❌ `No space left on device`

**Cause:** Docker is using too much disk space.

```bash
# Check disk usage
docker system df

# Clean up everything unused
docker system prune -a --volumes
```

---

### ❌ `Composer: out of memory`

**Cause:** PHP memory limit too low inside container.

```bash
sail composer install --no-dev
# or
docker compose exec app php -d memory_limit=-1 /usr/bin/composer install
```

---

### ❌ `.env` not found or app key not set

```bash
# Copy environment file
cp .env.example .env

# Generate app key
sail artisan key:generate
# or
docker compose exec app php artisan key:generate
```

---

### ❌ `MySQL connection refused`

Make sure your `.env` points to the **service name**, not `localhost`:

```env
DB_HOST=mysql        # ✅ correct — use the service name from docker-compose.yml
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=secret
```

---

## Quick Reference Cheat Sheet

```bash
# ─── CONTEXT ─────────────────────────────────────────────
docker context ls                          # List contexts
docker context use desktop-linux           # Switch to Docker Desktop
docker context use default                 # Switch to Docker Engine

# ─── PROJECT LIFECYCLE ────────────────────────────────────
sail up -d                                 # Start project (Sail, detached)
sail down                                  # Stop project (Sail)
docker compose up -d --build               # Start & rebuild (custom)
docker compose down                        # Stop project (custom)
docker compose down -v                     # Stop + delete volumes

# ─── CONTAINERS ───────────────────────────────────────────
docker ps                                  # Running containers
docker ps -a                               # All containers
docker stop <name>                         # Stop container
docker start <name>                        # Start container
docker rm <name>                           # Delete container
docker stats                               # Live resource usage

# ─── SHELL & COMMANDS ─────────────────────────────────────
sail bash                                  # Shell into app (Sail)
docker exec -it <name> bash                # Shell into any container
sail artisan <command>                     # Run Artisan (Sail)
docker compose exec app php artisan <cmd>  # Run Artisan (custom)
sail composer <command>                    # Run Composer (Sail)
sail npm <command>                         # Run NPM (Sail)

# ─── LOGS ─────────────────────────────────────────────────
sail logs -f                               # Live logs all (Sail)
docker compose logs -f <service>           # Live logs by service
docker logs -f <container>                 # Live logs by container
tail -f storage/logs/laravel.log           # Laravel app logs

# ─── CLEANUP ──────────────────────────────────────────────
docker image prune -a                      # Remove unused images
docker container prune                     # Remove stopped containers
docker volume prune                        # Remove unused volumes
docker system prune -a --volumes           # Remove EVERYTHING unused
```

---

## 🚀 Laravel Project Cheat Sheet

> Quick-access commands specifically for your day-to-day Laravel development inside Docker.

```bash
# ─── SETUP & INSTALL ──────────────────────────────────────
cp .env.example .env                                   # Create env file
sail artisan key:generate                              # Generate app key
sail artisan migrate                                   # Run all migrations
sail artisan migrate:fresh                             # Drop all tables & re-migrate
sail artisan migrate:fresh --seed                      # Fresh migrate + seed fake data
sail artisan db:seed                                   # Run seeders only
sail composer install                                  # Install PHP dependencies
sail npm install                                       # Install Node dependencies

# ─── ARTISAN GENERATORS ───────────────────────────────────
sail artisan make:controller UserController            # Create controller
sail artisan make:controller UserController --resource # Resource controller (CRUD)
sail artisan make:model Post -m                        # Model + migration
sail artisan make:model Post -mcr                      # Model + migration + controller
sail artisan make:migration create_posts_table         # Standalone migration
sail artisan make:seeder PostSeeder                    # Database seeder
sail artisan make:factory PostFactory                  # Model factory
sail artisan make:request StorePostRequest             # Form request (validation)
sail artisan make:middleware CheckRole                 # Middleware
sail artisan make:job SendEmailJob                     # Queued job
sail artisan make:event UserRegistered                 # Event
sail artisan make:listener SendWelcomeEmail            # Event listener
sail artisan make:command MyCustomCommand              # Custom Artisan command
sail artisan make:policy PostPolicy --model=Post       # Authorization policy

# ─── CACHE & CONFIG ───────────────────────────────────────
sail artisan cache:clear                               # Clear application cache
sail artisan config:clear                              # Clear config cache
sail artisan config:cache                              # Cache config for production
sail artisan route:clear                               # Clear route cache
sail artisan route:cache                               # Cache routes for production
sail artisan view:clear                                # Clear compiled views
sail artisan optimize                                  # Cache config + routes + views
sail artisan optimize:clear                            # Clear all caches at once

# ─── ROUTING & DEBUGGING ──────────────────────────────────
sail artisan route:list                                # List all registered routes
sail artisan route:list --name=user                    # Filter routes by name
sail artisan about                                     # Show app environment summary
sail artisan env                                       # Show current environment
sail php artisan tinker                                # Interactive Laravel REPL

# ─── DATABASE ─────────────────────────────────────────────
sail artisan migrate:status                            # Show migration status
sail artisan migrate:rollback                          # Rollback last migration batch
sail artisan migrate:rollback --step=3                 # Rollback last 3 batches
sail artisan migrate:reset                             # Rollback ALL migrations
sail artisan schema:dump                               # Dump schema to SQL file

# ─── QUEUES & JOBS ────────────────────────────────────────
sail artisan queue:work                                # Process queued jobs
sail artisan queue:work --tries=3                      # Retry failed jobs 3 times
sail artisan queue:failed                              # List failed jobs
sail artisan queue:retry all                           # Retry all failed jobs
sail artisan queue:flush                               # Delete all failed jobs

# ─── TESTING ──────────────────────────────────────────────
sail artisan test                                      # Run all tests
sail artisan test --filter=UserTest                    # Run specific test class
sail artisan test --filter=it_can_login                # Run specific test method
sail artisan test --coverage                           # Run tests with coverage report

# ─── COMPOSER ─────────────────────────────────────────────
sail composer require <package>                        # Add a package
sail composer require <package> --dev                  # Add a dev-only package
sail composer remove <package>                         # Remove a package
sail composer update                                   # Update all packages
sail composer dump-autoload                            # Regenerate autoloader

# ─── NPM / VITE ───────────────────────────────────────────
sail npm install                                       # Install all JS dependencies
sail npm run dev                                       # Start Vite dev server (hot reload)
sail npm run build                                     # Build assets for production
sail npm run preview                                   # Preview production build locally

# ─── STORAGE ──────────────────────────────────────────────
sail artisan storage:link                              # Create public storage symlink
```

> 💡 **Using custom docker-compose setup instead of Sail?**  
> Replace `sail` with `docker compose exec app` for every command above.  
> Example: `sail artisan migrate` → `docker compose exec app php artisan migrate`

---

## Part 10 — Controlling Your Laravel Project via Docker Desktop

> This section shows you how to manage every aspect of your Laravel Docker project using the **Docker Desktop GUI** — no terminal required for most day-to-day tasks.

---

### 10.1 — Make Sure Your Containers Are Visible in Docker Desktop

Before anything else, your terminal context must match Docker Desktop's context. Run this once:

```bash
docker context use desktop-linux
```

Then start your Laravel project from the terminal (just this once to bring it up):

```bash
# Sail
cd my-app && sail up -d

# Custom docker-compose
cd my-laravel-app && docker compose up -d
```

Once running, open Docker Desktop — you'll see your Laravel containers listed under the **Containers** tab.

---

### 10.2 — Starting & Stopping Your Laravel Project

In Docker Desktop → **Containers** tab, your project appears as a **group** (named after your project folder). It contains all your services: `app`, `nginx`, `mysql`, etc.

| Action | How to do it in Docker Desktop |
|--------|-------------------------------|
| **Start all containers** | Click the ▶️ **Play** button next to the project group name |
| **Stop all containers** | Click the ⏹ **Stop** button next to the project group name |
| **Restart all containers** | Click the 🔄 **Restart** button next to the project group name |
| **Start one container** | Expand the group → click ▶️ on that specific service |
| **Stop one container** | Expand the group → click ⏹ on that specific service |
| **Delete the project stack** | Click the 🗑️ **Delete** button → confirm to remove all containers |

> ⚠️ **Delete** only removes the containers — your code and database volumes are kept safe unless you explicitly delete volumes too.

---

### 10.3 — Opening a Terminal Inside a Container

You can run Artisan, Composer, and other commands directly from Docker Desktop without opening a separate terminal:

1. Go to **Containers** tab
2. Expand your project group
3. Click on the **`laravel.test`** (Sail) or **`laravel_app`** (custom) container
4. Click the **Terminal** tab (`>_` icon) at the top
5. A shell opens directly inside the container — run any command:

```bash
php artisan migrate
php artisan cache:clear
composer install
php artisan tinker
```

> 💡 This is the equivalent of running `sail bash` or `docker exec -it laravel_app bash` from your terminal.

---

### 10.4 — Viewing Laravel Logs in Real-Time

Docker Desktop has a built-in log viewer for each container:

1. **Containers** tab → expand your project group
2. Click on a container (e.g., `laravel.test` or `nginx`)
3. Click the **Logs** tab

You'll see a live stream of output. Use the search box to filter for specific keywords like `error`, `exception`, or `GET /api`.

**Which container logs what:**

| Container | What its logs show |
|-----------|-------------------|
| `laravel.test` / `app` | PHP errors, queue output, Artisan output |
| `nginx` | HTTP request logs (GET, POST, status codes) |
| `mysql` | Database queries and connection errors |
| `redis` | Cache hits, queue job events |

> 💡 For Laravel application-level logs (from `Log::info()`, exceptions, etc.), check `storage/logs/laravel.log` — either via the container terminal or your code editor.

---

### 10.5 — Monitoring CPU & Memory Usage

To see how many resources your Laravel project is consuming:

1. **Containers** tab → click on any container
2. Click the **Stats** tab

You'll see live graphs for:
- **CPU usage** — spikes during requests or queue jobs
- **Memory usage** — watch for leaks in long-running queue workers
- **Network I/O** — traffic between containers and the outside
- **Disk I/O** — database read/write activity

> 💡 If your MySQL container uses excessive memory, consider adding `--innodb-buffer-pool-size=128M` to your MySQL service config in `docker-compose.yml`.

---

### 10.6 — Inspecting & Managing the MySQL Database Volume

Your Laravel database data lives in a Docker **named volume** (e.g., `my-app_mysql`). To manage it:

1. Go to **Volumes** tab in Docker Desktop
2. Find your MySQL volume (named like `my-app_mysql` or `laravel_mysql_data`)
3. Click it to:
   - **Browse** the raw data files (for inspection)
   - **Delete** the volume (⚠️ this permanently destroys your database data — use only to reset)

**To reset your database cleanly:**

```bash
# From terminal
sail down -v          # Stop containers AND delete volumes
sail up -d            # Bring back up (fresh empty database)
sail artisan migrate --seed
```

---

### 10.7 — Managing Docker Images for Your Laravel Project

When you rebuild or update your project, old images can pile up. Clean them from Docker Desktop:

1. Go to **Images** tab
2. Look for images named like `my-app-laravel.test` or `laravel_app`
3. To **remove an old/unused image**: click the 🗑️ **Delete** button
4. To **rebuild a fresh image**: delete the old one, then run from terminal:

```bash
sail build --no-cache    # Sail
# or
docker compose up -d --build   # Custom
```

> 💡 You can also pull official images (like `mysql:8.0` or `nginx:alpine`) directly from the Images tab by clicking **Pull** and entering the image name.

---

### 10.8 — Adjusting Resources for Your Laravel Project

If your Laravel app feels slow or your MySQL is crashing, increase Docker's resource limits:

1. Click the **Settings (⚙️)** icon in Docker Desktop
2. Go to **Resources**

| Setting | Recommended for Laravel |
|---------|------------------------|
| **CPUs** | 2–4 cores |
| **Memory** | 4–8 GB (MySQL + PHP-FPM are memory-hungry) |
| **Swap** | 1–2 GB |
| **Disk image size** | 30–60 GB (images + volumes add up fast) |

3. Click **Apply & Restart** — Docker Desktop will restart with the new limits.

---

### 10.9 — Docker Desktop Laravel Workflow Summary

Here's the full day-to-day GUI workflow for working on your Laravel project:

```
1. Open Docker Desktop
2. Containers tab → click ▶️ on your project group  → Laravel is running
3. Open browser → http://localhost (or http://localhost:8080)
4. Make code changes in your editor (files are live-synced via volumes)
5. Need to run a migration?
      → Click your app container → Terminal tab → php artisan migrate
6. Something broken? Check logs
      → Click nginx or app container → Logs tab
7. Done for the day?
      → Containers tab → click ⏹ on your project group
```

> 💡 You don't need to keep Docker Desktop open while containers are running — they keep running in the background. Desktop is just for visibility and control.

---

> 💬 **Questions or issues?** Open an issue or reach out via [@johnboscocjt](https://github.com/johnboscocjt)  
> ⭐ If this guide helped you, consider starring the repo!# 🐳 Docker + Laravel on Ubuntu — Complete Developer Guide

> **Author:** [@johnboscocjt](https://github.com/johnboscocjt)  
> **Platform:** Ubuntu 20.04 / 22.04 / 24.04  
> **Covers:** Docker Engine · Docker Desktop · Laravel Projects · Day-to-Day Workflow

---

## 📑 Table of Contents

0. [⚠️ IMPORTANT — Read This First: Bandwidth & Data Usage](#️-important--read-this-first-bandwidth--data-usage)
1. [Prerequisites](#prerequisites)
2. [Part 1 — Install Docker Engine (CLI)](#part-1--install-docker-engine-cli)
3. [Part 2 — Install Docker Desktop (GUI)](#part-2--install-docker-desktop-gui)
4. [Part 3 — Understanding Docker Contexts](#part-3--understanding-docker-contexts)
5. [Part 4 — Start a New Laravel Project with Docker](#part-4--start-a-new-laravel-project-with-docker)
6. [Part 5 — Daily Workflow: Start, Stop, Restart](#part-5--daily-workflow-start-stop-restart)
7. [Part 6 — Running Commands Inside Containers](#part-6--running-commands-inside-containers)
8. [Part 7 — Viewing Logs & Monitoring](#part-7--viewing-logs--monitoring)
9. [Part 8 — Using Docker Desktop GUI](#part-8--using-docker-desktop-gui)
10. [Part 9 — Common Errors & Fixes](#part-9--common-errors--fixes)
11. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
12. [Part 10 — Controlling Your Laravel Project via Docker Desktop](#part-10--controlling-your-laravel-project-via-docker-desktop)

---

## ⚠️ IMPORTANT — Read This First: Bandwidth & Data Usage

> 🚨 **This section is critical if you are on a limited internet plan, mobile data, or a slow connection.**
> Docker downloads large files — sometimes several gigabytes — during setup. **Skipping or misusing commands can cause duplicate downloads that waste your data.**
> Read this section fully before running any command in this guide.

---

### 📦 How Much Data Will Docker Download?

Below is a breakdown of every major download this guide triggers, with approximate sizes. These are **one-time downloads** — Docker caches everything locally. You only re-download if you delete images or rebuild from scratch.

#### 🔧 Installation Downloads (One-Time)

| What | Approximate Size | When It Downloads |
|------|-----------------|-------------------|
| Docker Engine (apt packages) | ~100–150 MB | During `apt install docker-ce ...` |
| Docker Desktop `.deb` installer | ~550–600 MB | When you run `wget docker-desktop-amd64.deb` |
| Docker Desktop internal VM setup | ~300–500 MB | On very first launch of Docker Desktop |
| **Installation total** | **~1.0–1.3 GB** | First time only |

> 💡 If you only need the terminal (no GUI), **skip Docker Desktop entirely** and save ~1 GB. Docker Engine alone is sufficient for all Laravel development.

---

#### 🐘 Laravel Project Downloads (One-Time per Project)

These are downloaded the first time you create or start a Laravel project. Once pulled, Docker reuses the cached images.

| Image / Package | Approximate Size | Used For |
|----------------|-----------------|----------|
| `laravelsail/php84-composer` | ~1.5–2.0 GB | Main Laravel app container (PHP 8.4 + Composer + Node) |
| `mysql:8.0` | ~550 MB | Database |
| `redis:alpine` | ~35 MB | Cache / queues |
| `nginx:alpine` | ~40 MB | Web server (custom setup only) |
| `mailpit` | ~50 MB | Local email testing |
| Composer vendor packages | ~50–150 MB | PHP dependencies (`composer install`) |
| Node modules (`npm install`) | ~200–500 MB | Frontend JS dependencies |
| **Project total (Sail + mysql + redis + mailpit)** | **~2.5–3.5 GB** | First `sail up` only |

> ⚠️ **The Laravel Sail image (`laravelsail/php84-composer`) is the biggest single download** — approximately 1.5 to 2 GB on its own. This is because it bundles PHP, Composer, Node.js, npm, and many system libraries all in one image.

---

#### 🔁 What Triggers Re-Downloads (Data You Can Avoid Wasting)

These actions cause Docker to download things again. **Avoid them unless necessary.**

| Action | Re-downloads? | Approximate Data Lost | How to Avoid |
|--------|--------------|----------------------|--------------|
| `docker system prune -a --volumes` | ✅ Yes — everything | 2–4 GB | Only run when you truly want a full reset |
| `docker image prune -a` | ✅ Yes — all images | 2–3 GB | Only prune images you no longer need |
| `sail build --no-cache` | ✅ Yes — rebuilds layers | 500 MB–1.5 GB | Only use when a build is genuinely broken |
| `sail down -v` | ❌ No images, but deletes DB data | 0 MB (data loss only) | Use `sail down` instead to keep your database |
| `sail down` | ❌ No | 0 MB | ✅ Safe — images kept, containers just stop |
| `sail up -d` (subsequent runs) | ❌ No | 0 MB | ✅ Safe — reuses cached images every time |
| Failing `laravel.build` without `?php=84` | ⚠️ Partial waste | ~200–500 MB wasted | Always specify `?php=84` in the URL |

---

### 🧠 The Most Important Rules for Limited Data Users

#### Rule 1 — Always specify the PHP version when creating a project

```bash
# ❌ WRONG — will try PHP 8.5, fail, and waste ~200–500 MB on a broken pull
curl -s "https://laravel.build/my-app" | bash

# ✅ CORRECT — pulls the right image on the first attempt, no wasted data
curl -s "https://laravel.build/my-app?php=84" | bash
```

**What happens if you don't specify?** Laravel defaults to the latest PHP version (currently 8.5). Docker will attempt to pull `laravelsail/php85-composer`. If that image doesn't exist yet on Docker Hub, Docker downloads partial manifest and layer data (~200–500 MB depending on how far it gets), then fails with an error — and you've lost that data for nothing. You then have to run the command again with the correct version, downloading even more.

---

#### Rule 2 — Never use `--no-cache` or `prune` unless something is genuinely broken

```bash
# ❌ Wastes 500 MB–1.5 GB — forces a full image rebuild from scratch
sail build --no-cache

# ❌ Deletes ALL images — you re-download everything next time (2–4 GB)
docker system prune -a --volumes

# ✅ Safe restart — reuses all cached images, downloads nothing
sail down && sail up -d
```

---

#### Rule 3 — Use `sail down` NOT `sail down -v` for daily stops

```bash
# ❌ This deletes your database volume (data loss) — does NOT re-download images
#    but your database is gone and you must run migrations again
sail down -v

# ✅ This safely stops all containers and keeps your database intact
sail down
```

> `-v` stands for **volumes** — it destroys your MySQL data. Use it only when you intentionally want a clean database (e.g., switching branches with schema conflicts). It does not cause image re-downloads, but it means your next `sail artisan migrate` starts with an empty database.

---

#### Rule 4 — Once images are pulled, starting/stopping costs zero data

After the initial setup, your daily workflow costs **zero additional downloads**:

```bash
sail up -d             # 0 MB — starts from cached images
sail down              # 0 MB — stops containers, keeps everything
sail restart           # 0 MB — restarts from cache
sail artisan migrate   # 0 MB — runs inside existing container
```

Docker does not re-download images every time you start your project. Images are stored locally on your disk and reused indefinitely.

---

#### Rule 5 — Skip Docker Desktop if you are on a limited data plan

Docker Desktop adds **~1.0–1.1 GB** of downloads just for the installer and its internal VM. If you are comfortable with the terminal, **Docker Engine alone is everything you need.** All commands in this guide work without Docker Desktop.

Only install Docker Desktop if you specifically need the visual GUI for logs, stats, or container management.

---

### 📊 Complete Data Budget Summary

| Scenario | Total Download | Notes |
|----------|---------------|-------|
| Docker Engine only (no Desktop) | ~100–150 MB | Minimal setup, terminal only |
| Docker Engine + Docker Desktop | ~1.0–1.3 GB | Adds visual GUI |
| First Laravel project (Sail + mysql + redis) | ~2.5–3.0 GB | One-time image pull |
| First Laravel project (custom nginx setup) | ~700 MB–1.0 GB | Smaller images, no Sail |
| Daily use after setup | **0 MB** | Everything cached locally |
| Full reset (`system prune -a`) | Re-downloads 2–4 GB | Avoid unless truly needed |
| Failed `laravel.build` (no PHP version) | ~200–500 MB wasted | Always use `?php=84` |

> 🏁 **Bottom line:** Expect to spend **3–5 GB total** on the very first setup. After that, your daily usage is **0 MB** of additional downloads. Protect that initial investment — don't prune images unless you have a specific reason.

---

### ⚖️ Bandwidth Comparison: Traditional Laravel vs Docker

> This is one of the most important decisions you'll make. Here is an honest, side-by-side breakdown of what each approach actually downloads so you can choose what's right for your internet situation.

---

#### 🛤️ Method 1 — Traditional Laravel (No Docker)

This is the classic way: install PHP, Composer, MySQL, and Node directly on your machine using your system's package manager.

**What gets installed and how much data it costs:**

| Step | Command | Approximate Size | What It Installs |
|------|---------|-----------------|------------------|
| 1. Install PHP + extensions | `sudo apt install php8.4 php8.4-mbstring php8.4-xml php8.4-curl php8.4-mysql ...` | ~50–80 MB | PHP runtime + required extensions |
| 2. Install Composer | `curl ... \| php -- --install-dir=...` | ~2–5 MB | PHP package manager |
| 3. Create Laravel project | `composer create-project laravel/laravel my-app` | ~50–80 MB | Laravel framework + all PHP dependencies |
| 4. Install MySQL | `sudo apt install mysql-server` | ~200–250 MB | MySQL database server |
| 5. Install Node.js + npm | `sudo apt install nodejs npm` | ~70–100 MB | JavaScript runtime |
| 6. Install frontend deps | `npm install` (inside project) | ~150–300 MB | Vite, Tailwind, etc. |
| **Total (first time)** | | **~520–815 MB** | Full working Laravel environment |

**Subsequent projects on the same machine:**

| Step | Cost | Reason |
|------|------|--------|
| PHP, MySQL, Node already installed | 0 MB | System-wide, reused by every project |
| `composer create-project laravel/laravel my-app2` | ~30–50 MB | Only downloads packages not already in Composer cache |
| `npm install` | ~50–150 MB | Some packages cached by npm, some new |
| **Total for 2nd project** | **~80–200 MB** | Much cheaper after first setup |

---

#### 🐳 Method 2 — Laravel with Docker (Sail)

Docker packages each project's entire environment into isolated containers. Every service (PHP, MySQL, Redis, Node) runs inside its own image.

**What gets downloaded and how much data it costs:**

| Step | Command | Approximate Size | What It Downloads |
|------|---------|-----------------|------------------|
| 1. Install Docker Engine | `apt install docker-ce ...` | ~100–150 MB | Docker daemon + CLI tools |
| 2. (Optional) Docker Desktop | `apt install docker-desktop-amd64.deb` | ~850 MB–1.1 GB | GUI + internal VM |
| 3. Pull Sail image | Auto on first `sail up` | ~1.5–2.0 GB | PHP 8.4 + Composer + Node + npm + extensions |
| 4. Pull MySQL image | Auto on first `sail up` | ~550 MB | Full MySQL 8.0 server |
| 5. Pull Redis image | Auto on first `sail up` | ~35 MB | Redis server |
| 6. Pull Mailpit image | Auto on first `sail up` | ~50 MB | Email testing server |
| 7. Composer install | `sail composer install` | ~50–80 MB | PHP packages (runs inside container) |
| 8. npm install | `sail npm install` | ~150–300 MB | Frontend packages (runs inside container) |
| **Total (Engine + Sail + all services)** | | **~2.6–3.4 GB** | Full Docker Laravel stack |
| **Total (Engine only, no Desktop)** | | **~2.1–2.8 GB** | Same stack, no GUI |

**Subsequent projects on the same machine:**

| Step | Cost | Reason |
|------|------|--------|
| Docker Engine already installed | 0 MB | Installed once, used forever |
| Sail image already pulled | 0 MB | Same PHP version = same cached image |
| MySQL image already pulled | 0 MB | `mysql:8.0` cached from first project |
| Redis / Mailpit already pulled | 0 MB | Cached |
| `composer install` | ~30–50 MB | Only new/changed packages |
| `npm install` | ~50–150 MB | Some packages cached |
| **Total for 2nd project** | **~80–200 MB** | Same as traditional — images are reused |

> 💡 After the first Docker project, subsequent projects are just as cheap as traditional Laravel — because all the heavy images are already cached on your machine.

---

#### 📊 Head-to-Head Comparison Table

| | Traditional Laravel | Docker (Sail, Engine only) | Docker (Sail + Desktop) |
|--|--------------------|-----------------------------|------------------------|
| **First project download** | ~520–815 MB | ~2.1–2.8 GB | ~2.6–3.4 GB |
| **Second project download** | ~80–200 MB | ~80–200 MB | ~80–200 MB |
| **Daily start/stop cost** | 0 MB | 0 MB | 0 MB |
| **PHP version conflicts** | ⚠️ Risk (system-wide PHP) | ✅ None (isolated per project) |✅ None |
| **MySQL conflicts** | ⚠️ Risk (one system MySQL) | ✅ None (each project isolated) | ✅ None |
| **Works on teammate's machine** | ⚠️ "Works on my machine" risk | ✅ Identical environment | ✅ Identical environment |
| **Wipe and reset a project** | ❌ Manual and messy | ✅ `sail down -v && sail up -d` | ✅ Same |
| **Good for limited internet** | ✅ Yes — much lighter first setup | ⚠️ Only after initial large download | ❌ High first cost |
| **Disk space used** | ~800 MB–1.5 GB | ~4–6 GB (images + volumes) | ~5–7 GB |

---

#### 🤔 Which Should You Choose?

**Choose Traditional Laravel if:**
- You are on a severely limited data plan and this is your first time setting up
- You only run one Laravel version at a time
- You are a solo developer and don't need to match teammates' environments
- You're comfortable managing PHP, MySQL, and Node directly on your system

```bash
# Traditional setup (lighter on data)
sudo apt install php8.4 php8.4-cli php8.4-mbstring php8.4-xml php8.4-curl php8.4-mysql php8.4-zip
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer create-project laravel/laravel my-app
cd my-app && php artisan serve
# Open http://localhost:8000
```

**Total first-time cost: ~520–815 MB**

---

**Choose Docker (Sail) if:**
- You have a decent data allowance for the initial setup (3–4 GB)
- You work on multiple Laravel projects that may need different PHP versions
- You work in a team and need everyone's environment to match exactly
- You want a clean, disposable environment with no risk of polluting your system

```bash
# Docker setup (heavier upfront, but better isolation)
curl -s "https://laravel.build/my-app?php=84" | bash
cd my-app && sail up -d
# Open http://localhost
```

**Total first-time cost: ~2.1–3.4 GB**

---

#### 💾 `laravel new` vs `composer create-project` — Are They Different?

Many developers wonder if `laravel new` and `composer create-project laravel/laravel` use different amounts of data. Here's the truth:

| Method | Extra Download | Notes |
|--------|---------------|-------|
| `composer create-project laravel/laravel my-app` | 0 MB extra | Downloads Laravel directly via Composer — no extra tools |
| `laravel new my-app` (Laravel Installer) | ~2–5 MB extra (one-time) | Requires installing the Laravel Installer globally first: `composer global require laravel/installer` |
| **The actual Laravel project download** | **~50–80 MB** | **Same for both methods** — same packages, same size |

```bash
# Method A — No extra tools needed (~50–80 MB total)
composer create-project laravel/laravel my-app

# Method B — Install Laravel Installer once, then use it forever (~2 MB extra, one-time)
composer global require laravel/installer   # one-time, ~2–5 MB
laravel new my-app                          # ~50–80 MB each time (same as Method A)
```

> ✅ **Both methods produce the exact same Laravel project** with the same file size and the same dependencies. `laravel new` is just a convenience wrapper — it does not download anything extra for the project itself. The only difference is the one-time install of the Laravel Installer tool (~2–5 MB).

---

#### 📉 Visual Summary: Data Cost Over 3 Projects

```
Traditional Laravel
────────────────────────────────────────────────────────────
Project 1:  ████████████████████████████████  ~700 MB
Project 2:  ████  ~120 MB
Project 3:  ████  ~120 MB
Total:      ████████████████████████████████████████  ~940 MB

Docker (Sail) — Engine Only
────────────────────────────────────────────────────────────
Project 1:  ████████████████████████████████████████████████████████████████  ~2.5 GB
Project 2:  ████  ~120 MB
Project 3:  ████  ~120 MB
Total:      ████████████████████████████████████████████████████████████████████  ~2.74 GB

Docker (Sail) — With Docker Desktop
────────────────────────────────────────────────────────────
Project 1:  ██████████████████████████████████████████████████████████████████████████  ~3.2 GB
Project 2:  ████  ~120 MB
Project 3:  ████  ~120 MB
Total:      ████████████████████████████████████████████████████████████████████████████  ~3.44 GB
```

> 🔑 **Key takeaway:** Traditional Laravel is significantly cheaper for your first project. By the third project, the gap narrows because Docker reuses all its cached images. If you plan to build many Laravel projects over time, Docker's upfront cost pays for itself in consistency and isolation.

---

## Prerequisites

Before starting, make sure your Ubuntu system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

Check your Ubuntu version:

```bash
cat /etc/os-release
```

You should be on **Ubuntu 20.04, 22.04, or 24.04** (Desktop edition — not Server — if you want Docker Desktop).

---

## Part 1 — Install Docker Engine (CLI)

> Docker Engine is the core Docker daemon that runs in the background. Always install this first — Docker Desktop depends on it too.

### Step 1.1 — Remove old/conflicting Docker packages

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove -y $pkg
done
```

### Step 1.2 — Install required dependencies

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### Step 1.3 — Add Docker's official GPG key

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Step 1.4 — Add Docker's repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 1.5 — Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Step 1.6 — Add your user to the docker group (avoid using `sudo` every time)

```bash
sudo usermod -aG docker $USER
newgrp docker
```

> ⚠️ Log out and back in (or reboot) for this to fully take effect.

### Step 1.7 — Verify Docker Engine is working

```bash
docker --version
docker compose version
docker run hello-world
```

✅ You should see `Hello from Docker!` — Docker Engine is installed and working.

### Step 1.8 — Enable Docker to start on boot

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

## Part 2 — Install Docker Desktop (GUI)

> Docker Desktop gives you a visual dashboard to manage containers, images, and volumes. It runs its **own internal VM** separate from Docker Engine.

### Step 2.1 — Install GNOME Terminal (required dependency)

```bash
sudo apt update
sudo apt install -y gnome-terminal
```

### Step 2.2 — Download Docker Desktop `.deb` package

```bash
wget https://desktop.docker.com/linux/main/amd64/docker-desktop-amd64.deb
```

> 💡 This downloads the latest stable release directly from Docker.

### Step 2.3 — Install Docker Desktop

```bash
sudo apt install ./docker-desktop-amd64.deb
```

If you see dependency errors, fix them with:

```bash
sudo apt --fix-broken install
```

### Step 2.4 — Launch Docker Desktop

**Option A — From your Applications Menu (recommended):**  
Search for **"Docker Desktop"** in your app launcher and click it.

**Option B — From Terminal:**

```bash
systemctl --user start docker-desktop
```

> The first launch may take a few minutes as it sets up the internal VM. You'll see the 🐳 Docker whale icon in your system tray when it's ready.

### Step 2.5 — Verify Docker Desktop is running

```bash
docker --version
docker context ls
```

You should see `desktop-linux` listed as one of the contexts.

---

## Part 3 — Understanding Docker Contexts

> **This is the most important concept when using both Docker Engine and Docker Desktop on the same machine.**

Docker Engine and Docker Desktop use **separate daemons** (separate backends). Containers you create in one context **won't appear** in the other.

### View all contexts

```bash
docker context ls
```

Output example:

```
NAME              DESCRIPTION                               DOCKER ENDPOINT
default           Current DOCKER_HOST based configuration   unix:///var/run/docker.sock
desktop-linux *   Docker Desktop                            ...
```

The `*` shows the currently active context.

### Switch to Docker Desktop context

```bash
docker context use desktop-linux
```

### Switch back to Docker Engine (native)

```bash
docker context use default
```

### 💡 Recommendation

Pick **one context** for your Laravel projects and stick with it. Most developers use:
- `default` → if you work mostly from the terminal
- `desktop-linux` → if you prefer the visual Docker Desktop dashboard

---

## Part 4 — Start a New Laravel Project with Docker

There are two common approaches. **Option A** (Laravel Sail) is the easiest for beginners.

---

### Option A — Laravel Sail (Recommended for Beginners)

Laravel Sail is Laravel's official Docker development environment.

#### Step 4A.1 — Create a new Laravel project using Sail

Start with the basic command — no extra options needed:

```bash
curl -s "https://laravel.build/my-app" | bash
```

Replace `my-app` with your project name. You can run this **from any folder** — it will create the project as a subfolder wherever your terminal currently is.

> #### ⚠️ If you see this error:
> ```
> docker: Error response from daemon: pull access denied for laravelsail/php85-composer,
> repository does not exist or may require 'docker login'
> ```
> **What happened:** `laravel.build` automatically picks the latest PHP version (e.g. PHP 8.5). If a Sail Docker image for that version hasn't been published yet — because it's too new — Docker can't pull it and the whole setup fails. Your project folder either won't be created or will be left empty and broken.
>
> **The fix:** Specify a PHP version that actually has a published image:
>
> ```bash
> # Remove the broken folder first (if it was partially created)
> rm -rf my-app
>
> # Re-run with a specific PHP version
> curl -s "https://laravel.build/my-app?php=84" | bash
> ```
>
> **Available PHP versions:**
>
> | PHP Version | URL Parameter |
> |-------------|---------------|
> | PHP 8.4 ✅ (recommended) | `?php=84` |
> | PHP 8.3 | `?php=83` |
> | PHP 8.2 | `?php=82` |

To include specific services (e.g., MySQL, Redis, Mailpit):

```bash
curl -s "https://laravel.build/my-app?with=mysql,redis,mailpit&php=84" | bash
```

#### Step 4A.2 — Enter the project directory

```bash
cd my-app
```

#### Step 4A.3 — Start the project

```bash
./vendor/bin/sail up
```

Or run in the background (detached mode):

```bash
./vendor/bin/sail up -d
```

#### Step 4A.4 — Create a shell alias for convenience

Instead of typing `./vendor/bin/sail` every time, add this alias to your shell:

```bash
echo "alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'" >> ~/.bashrc
source ~/.bashrc
```

Now you can just type `sail` instead of `./vendor/bin/sail`.

#### Step 4A.5 — Open your app in the browser

```
http://localhost
```

---

### Option B — Custom `docker-compose.yml` (Full Control)

Use this approach when you need custom configuration or are integrating Docker into an existing project.

#### Step 4B.1 — Create your project directory

```bash
mkdir my-laravel-app && cd my-laravel-app
```

#### Step 4B.2 — Create a `docker-compose.yml` file

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
version: "3.8"

services:

  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: laravel_app
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - .:/var/www
    networks:
      - laravel

  nginx:
    image: nginx:alpine
    container_name: laravel_nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - .:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    networks:
      - laravel

  mysql:
    image: mysql:8.0
    container_name: laravel_mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: root
      MYSQL_PASSWORD: secret
      MYSQL_USER: laravel
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - laravel

networks:
  laravel:
    driver: bridge

volumes:
  mysql_data:
```

#### Step 4B.3 — Create a `Dockerfile`

```bash
nano Dockerfile
```

```dockerfile
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    nodejs \
    npm

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

COPY . .

RUN composer install --no-scripts --no-autoloader

RUN composer dump-autoload

EXPOSE 9000
CMD ["php-fpm"]
```

#### Step 4B.4 — Create the Nginx config

```bash
mkdir -p docker/nginx
nano docker/nginx/default.conf
```

```nginx
server {
    listen 80;
    index index.php index.html;
    root /var/www/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

#### Step 4B.5 — Create a new Laravel app inside the container

```bash
docker compose run --rm app composer create-project laravel/laravel .
```

#### Step 4B.6 — Build and start everything

```bash
docker compose up -d --build
```

#### Step 4B.7 — Open your app

```
http://localhost:8080
```

---

## Part 5 — Daily Workflow: Start, Stop, Restart

### Using Laravel Sail

| Action | Command |
|--------|---------|
| Start containers | `sail up -d` |
| Stop containers | `sail down` |
| Restart containers | `sail restart` |
| Stop & remove volumes | `sail down -v` |
| Rebuild containers | `sail build --no-cache` |
| View running containers | `sail ps` |

### Using Docker Compose (custom setup)

| Action | Command |
|--------|---------|
| Start containers | `docker compose up -d` |
| Stop containers | `docker compose down` |
| Restart containers | `docker compose restart` |
| Stop & remove volumes | `docker compose down -v` |
| Rebuild containers | `docker compose up -d --build` |
| View running containers | `docker compose ps` |

### General Docker Commands

```bash
# List all running containers
docker ps

# List all containers (including stopped ones)
docker ps -a

# List all images
docker images

# Stop a specific container
docker stop <container_name_or_id>

# Start a stopped container
docker start <container_name_or_id>

# Remove a stopped container
docker rm <container_name_or_id>

# Remove an image
docker rmi <image_name_or_id>

# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune -a

# Remove everything (containers, images, volumes, networks) — USE WITH CAUTION
docker system prune -a --volumes
```

---

## Part 6 — Running Commands Inside Containers

### Using Laravel Sail

```bash
# Run Artisan commands
sail artisan migrate
sail artisan make:controller UserController
sail artisan make:model Post -m
sail artisan cache:clear
sail artisan config:clear
sail artisan route:list

# Composer
sail composer install
sail composer require laravel/sanctum
sail composer update

# NPM / Node
sail npm install
sail npm run dev
sail npm run build

# Open a bash shell inside the app container
sail bash

# Run PHP directly
sail php --version
sail php artisan tinker

# Run tests
sail test
sail artisan test
```

### Using Docker Compose (custom setup)

```bash
# Enter the app container shell
docker compose exec app bash

# Or for a one-off command without entering the shell:
docker compose exec app php artisan migrate
docker compose exec app php artisan make:controller UserController
docker compose exec app composer install
docker compose exec app php artisan cache:clear

# Run a command in the MySQL container
docker compose exec mysql mysql -u laravel -psecret laravel
```

### Using `docker exec` directly

```bash
# Enter a running container by name
docker exec -it laravel_app bash

# Run a single command
docker exec laravel_app php artisan migrate

# Run as a specific user
docker exec -it --user www-data laravel_app bash
```

---

## Part 7 — Viewing Logs & Monitoring

### View container logs

```bash
# View logs for all services (Sail)
sail logs

# Follow logs in real-time (Sail)
sail logs -f

# View logs for a specific service (Sail)
sail logs nginx
sail logs mysql

# Docker Compose — all services
docker compose logs

# Docker Compose — follow all logs
docker compose logs -f

# Docker Compose — specific service
docker compose logs -f app
docker compose logs -f nginx
docker compose logs -f mysql

# Native Docker — view logs by container name
docker logs laravel_app
docker logs -f laravel_app          # follow (live stream)
docker logs --tail=100 laravel_app  # last 100 lines only
```

### Laravel application logs

```bash
# View Laravel logs from inside the project
tail -f storage/logs/laravel.log

# Or from inside the container
docker compose exec app tail -f storage/logs/laravel.log
```

### Monitor resource usage

```bash
# Live CPU/memory/network stats for all running containers
docker stats

# Stats for a specific container
docker stats laravel_app

# View detailed container info
docker inspect laravel_app
```

### Check container status

```bash
# Running containers only
docker ps

# All containers (running + stopped)
docker ps -a

# With disk size info
docker ps --size
```

---

## Part 8 — Using Docker Desktop GUI

Docker Desktop provides a visual interface for everything you've been doing in the terminal.

### Starting Docker Desktop

- Search **"Docker Desktop"** in your application launcher and click it, OR
- Run from terminal: `systemctl --user start docker-desktop`

Wait for the 🐳 whale icon to appear in your system tray — that means it's running.

### Containers Tab

- See **all running and stopped containers** at a glance
- Click a container to see its **logs in real-time**
- Use the ▶️ / ⏹ / 🔄 buttons to **start, stop, or restart** a container
- Click the **Terminal** icon `>_` to open a shell directly inside the container
- Click the **Stats** tab to see **CPU and memory usage**

### Images Tab

- View all downloaded Docker images
- See image **size and creation date**
- Delete images you no longer need with the 🗑️ button
- **Pull** new images by clicking "Pull"

### Volumes Tab

- View all named volumes (like your MySQL database data)
- Inspect volume contents
- Delete volumes you no longer need

### Dev Environments Tab

- Create shareable dev environments (useful for teams)

### Settings (Gear Icon ⚙️)

| Setting | What it does |
|---------|-------------|
| Resources → CPU | Limit how many CPU cores Docker Desktop VM uses |
| Resources → Memory | Set max RAM (default 2GB — increase for larger apps) |
| Resources → Disk | Set how much disk space Docker can use |
| General → Start on login | Auto-start Docker Desktop when you log in |
| Docker Engine | Edit the raw Docker daemon JSON config |

### Switching contexts in Docker Desktop

Docker Desktop defaults to the `desktop-linux` context. To make sure you're managing the right containers, always match your terminal context:

```bash
# To align terminal with Docker Desktop
docker context use desktop-linux

# To use native Docker Engine (containers won't show in Desktop)
docker context use default
```

---

## Part 9 — Common Errors & Fixes

### ❌ `pull access denied for laravelsail/php85-composer` (or any php8x)

**Cause:** Running `curl -s "https://laravel.build/my-app" | bash` without specifying a PHP version causes Laravel to default to the latest PHP (e.g. 8.5), which may not have a Sail Docker image published yet. The project folder gets created but is empty or broken.

**Fix:** Always pass a valid `?php=` version in the URL:

```bash
# ✅ Correct — specify PHP 8.4 explicitly
curl -s "https://laravel.build/my-app?php=84" | bash

# ✅ With services too
curl -s "https://laravel.build/my-app?with=mysql,redis,mailpit&php=84" | bash
```

If the folder was partially created from a failed attempt, remove it first:

```bash
rm -rf my-app
curl -s "https://laravel.build/my-app?php=84" | bash
```

**Valid PHP options:** `php=82`, `php=83`, `php=84`

---

### ❌ `docker: permission denied`

**Cause:** Your user isn't in the `docker` group.

```bash
sudo usermod -aG docker $USER
newgrp docker
# Then log out and back in
```

---

### ❌ `Port is already allocated` or `bind: address already in use`

**Cause:** Something else is using the port (e.g., port 80 or 3306).

```bash
# Find what's using port 80
sudo lsof -i :80
sudo lsof -i :3306

# Kill the process using the port
sudo kill -9 <PID>

# Or change the port in docker-compose.yml
ports:
  - "8081:80"   # change 8080 to something else
```

---

### ❌ `Cannot connect to the Docker daemon`

**Cause:** Docker isn't running.

```bash
# Start Docker Engine
sudo systemctl start docker

# Start Docker Desktop
systemctl --user start docker-desktop
```

---

### ❌ `Sail: command not found`

**Cause:** Shell alias not set up.

```bash
echo "alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'" >> ~/.bashrc
source ~/.bashrc
```

---

### ❌ `No space left on device`

**Cause:** Docker is using too much disk space.

```bash
# Check disk usage
docker system df

# Clean up everything unused
docker system prune -a --volumes
```

---

### ❌ `Composer: out of memory`

**Cause:** PHP memory limit too low inside container.

```bash
sail composer install --no-dev
# or
docker compose exec app php -d memory_limit=-1 /usr/bin/composer install
```

---

### ❌ `.env` not found or app key not set

```bash
# Copy environment file
cp .env.example .env

# Generate app key
sail artisan key:generate
# or
docker compose exec app php artisan key:generate
```

---

### ❌ `MySQL connection refused`

Make sure your `.env` points to the **service name**, not `localhost`:

```env
DB_HOST=mysql        # ✅ correct — use the service name from docker-compose.yml
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=secret
```

---

## Quick Reference Cheat Sheet

```bash
# ─── CONTEXT ─────────────────────────────────────────────
docker context ls                          # List contexts
docker context use desktop-linux           # Switch to Docker Desktop
docker context use default                 # Switch to Docker Engine

# ─── PROJECT LIFECYCLE ────────────────────────────────────
sail up -d                                 # Start project (Sail, detached)
sail down                                  # Stop project (Sail)
docker compose up -d --build               # Start & rebuild (custom)
docker compose down                        # Stop project (custom)
docker compose down -v                     # Stop + delete volumes

# ─── CONTAINERS ───────────────────────────────────────────
docker ps                                  # Running containers
docker ps -a                               # All containers
docker stop <name>                         # Stop container
docker start <name>                        # Start container
docker rm <name>                           # Delete container
docker stats                               # Live resource usage

# ─── SHELL & COMMANDS ─────────────────────────────────────
sail bash                                  # Shell into app (Sail)
docker exec -it <name> bash                # Shell into any container
sail artisan <command>                     # Run Artisan (Sail)
docker compose exec app php artisan <cmd>  # Run Artisan (custom)
sail composer <command>                    # Run Composer (Sail)
sail npm <command>                         # Run NPM (Sail)

# ─── LOGS ─────────────────────────────────────────────────
sail logs -f                               # Live logs all (Sail)
docker compose logs -f <service>           # Live logs by service
docker logs -f <container>                 # Live logs by container
tail -f storage/logs/laravel.log           # Laravel app logs

# ─── CLEANUP ──────────────────────────────────────────────
docker image prune -a                      # Remove unused images
docker container prune                     # Remove stopped containers
docker volume prune                        # Remove unused volumes
docker system prune -a --volumes           # Remove EVERYTHING unused
```

---

## 🚀 Laravel Project Cheat Sheet

> Quick-access commands specifically for your day-to-day Laravel development inside Docker.

```bash
# ─── SETUP & INSTALL ──────────────────────────────────────
cp .env.example .env                                   # Create env file
sail artisan key:generate                              # Generate app key
sail artisan migrate                                   # Run all migrations
sail artisan migrate:fresh                             # Drop all tables & re-migrate
sail artisan migrate:fresh --seed                      # Fresh migrate + seed fake data
sail artisan db:seed                                   # Run seeders only
sail composer install                                  # Install PHP dependencies
sail npm install                                       # Install Node dependencies

# ─── ARTISAN GENERATORS ───────────────────────────────────
sail artisan make:controller UserController            # Create controller
sail artisan make:controller UserController --resource # Resource controller (CRUD)
sail artisan make:model Post -m                        # Model + migration
sail artisan make:model Post -mcr                      # Model + migration + controller
sail artisan make:migration create_posts_table         # Standalone migration
sail artisan make:seeder PostSeeder                    # Database seeder
sail artisan make:factory PostFactory                  # Model factory
sail artisan make:request StorePostRequest             # Form request (validation)
sail artisan make:middleware CheckRole                 # Middleware
sail artisan make:job SendEmailJob                     # Queued job
sail artisan make:event UserRegistered                 # Event
sail artisan make:listener SendWelcomeEmail            # Event listener
sail artisan make:command MyCustomCommand              # Custom Artisan command
sail artisan make:policy PostPolicy --model=Post       # Authorization policy

# ─── CACHE & CONFIG ───────────────────────────────────────
sail artisan cache:clear                               # Clear application cache
sail artisan config:clear                              # Clear config cache
sail artisan config:cache                              # Cache config for production
sail artisan route:clear                               # Clear route cache
sail artisan route:cache                               # Cache routes for production
sail artisan view:clear                                # Clear compiled views
sail artisan optimize                                  # Cache config + routes + views
sail artisan optimize:clear                            # Clear all caches at once

# ─── ROUTING & DEBUGGING ──────────────────────────────────
sail artisan route:list                                # List all registered routes
sail artisan route:list --name=user                    # Filter routes by name
sail artisan about                                     # Show app environment summary
sail artisan env                                       # Show current environment
sail php artisan tinker                                # Interactive Laravel REPL

# ─── DATABASE ─────────────────────────────────────────────
sail artisan migrate:status                            # Show migration status
sail artisan migrate:rollback                          # Rollback last migration batch
sail artisan migrate:rollback --step=3                 # Rollback last 3 batches
sail artisan migrate:reset                             # Rollback ALL migrations
sail artisan schema:dump                               # Dump schema to SQL file

# ─── QUEUES & JOBS ────────────────────────────────────────
sail artisan queue:work                                # Process queued jobs
sail artisan queue:work --tries=3                      # Retry failed jobs 3 times
sail artisan queue:failed                              # List failed jobs
sail artisan queue:retry all                           # Retry all failed jobs
sail artisan queue:flush                               # Delete all failed jobs

# ─── TESTING ──────────────────────────────────────────────
sail artisan test                                      # Run all tests
sail artisan test --filter=UserTest                    # Run specific test class
sail artisan test --filter=it_can_login                # Run specific test method
sail artisan test --coverage                           # Run tests with coverage report

# ─── COMPOSER ─────────────────────────────────────────────
sail composer require <package>                        # Add a package
sail composer require <package> --dev                  # Add a dev-only package
sail composer remove <package>                         # Remove a package
sail composer update                                   # Update all packages
sail composer dump-autoload                            # Regenerate autoloader

# ─── NPM / VITE ───────────────────────────────────────────
sail npm install                                       # Install all JS dependencies
sail npm run dev                                       # Start Vite dev server (hot reload)
sail npm run build                                     # Build assets for production
sail npm run preview                                   # Preview production build locally

# ─── STORAGE ──────────────────────────────────────────────
sail artisan storage:link                              # Create public storage symlink
```

> 💡 **Using custom docker-compose setup instead of Sail?**  
> Replace `sail` with `docker compose exec app` for every command above.  
> Example: `sail artisan migrate` → `docker compose exec app php artisan migrate`

---

## Part 10 — Controlling Your Laravel Project via Docker Desktop

> This section shows you how to manage every aspect of your Laravel Docker project using the **Docker Desktop GUI** — no terminal required for most day-to-day tasks.

---

### 10.1 — Make Sure Your Containers Are Visible in Docker Desktop

Before anything else, your terminal context must match Docker Desktop's context. Run this once:

```bash
docker context use desktop-linux
```

Then start your Laravel project from the terminal (just this once to bring it up):

```bash
# Sail
cd my-app && sail up -d

# Custom docker-compose
cd my-laravel-app && docker compose up -d
```

Once running, open Docker Desktop — you'll see your Laravel containers listed under the **Containers** tab.

---

### 10.2 — Starting & Stopping Your Laravel Project

In Docker Desktop → **Containers** tab, your project appears as a **group** (named after your project folder). It contains all your services: `app`, `nginx`, `mysql`, etc.

| Action | How to do it in Docker Desktop |
|--------|-------------------------------|
| **Start all containers** | Click the ▶️ **Play** button next to the project group name |
| **Stop all containers** | Click the ⏹ **Stop** button next to the project group name |
| **Restart all containers** | Click the 🔄 **Restart** button next to the project group name |
| **Start one container** | Expand the group → click ▶️ on that specific service |
| **Stop one container** | Expand the group → click ⏹ on that specific service |
| **Delete the project stack** | Click the 🗑️ **Delete** button → confirm to remove all containers |

> ⚠️ **Delete** only removes the containers — your code and database volumes are kept safe unless you explicitly delete volumes too.

---

### 10.3 — Opening a Terminal Inside a Container

You can run Artisan, Composer, and other commands directly from Docker Desktop without opening a separate terminal:

1. Go to **Containers** tab
2. Expand your project group
3. Click on the **`laravel.test`** (Sail) or **`laravel_app`** (custom) container
4. Click the **Terminal** tab (`>_` icon) at the top
5. A shell opens directly inside the container — run any command:

```bash
php artisan migrate
php artisan cache:clear
composer install
php artisan tinker
```

> 💡 This is the equivalent of running `sail bash` or `docker exec -it laravel_app bash` from your terminal.

---

### 10.4 — Viewing Laravel Logs in Real-Time

Docker Desktop has a built-in log viewer for each container:

1. **Containers** tab → expand your project group
2. Click on a container (e.g., `laravel.test` or `nginx`)
3. Click the **Logs** tab

You'll see a live stream of output. Use the search box to filter for specific keywords like `error`, `exception`, or `GET /api`.

**Which container logs what:**

| Container | What its logs show |
|-----------|-------------------|
| `laravel.test` / `app` | PHP errors, queue output, Artisan output |
| `nginx` | HTTP request logs (GET, POST, status codes) |
| `mysql` | Database queries and connection errors |
| `redis` | Cache hits, queue job events |

> 💡 For Laravel application-level logs (from `Log::info()`, exceptions, etc.), check `storage/logs/laravel.log` — either via the container terminal or your code editor.

---

### 10.5 — Monitoring CPU & Memory Usage

To see how many resources your Laravel project is consuming:

1. **Containers** tab → click on any container
2. Click the **Stats** tab

You'll see live graphs for:
- **CPU usage** — spikes during requests or queue jobs
- **Memory usage** — watch for leaks in long-running queue workers
- **Network I/O** — traffic between containers and the outside
- **Disk I/O** — database read/write activity

> 💡 If your MySQL container uses excessive memory, consider adding `--innodb-buffer-pool-size=128M` to your MySQL service config in `docker-compose.yml`.

---

### 10.6 — Inspecting & Managing the MySQL Database Volume

Your Laravel database data lives in a Docker **named volume** (e.g., `my-app_mysql`). To manage it:

1. Go to **Volumes** tab in Docker Desktop
2. Find your MySQL volume (named like `my-app_mysql` or `laravel_mysql_data`)
3. Click it to:
   - **Browse** the raw data files (for inspection)
   - **Delete** the volume (⚠️ this permanently destroys your database data — use only to reset)

**To reset your database cleanly:**

```bash
# From terminal
sail down -v          # Stop containers AND delete volumes
sail up -d            # Bring back up (fresh empty database)
sail artisan migrate --seed
```

---

### 10.7 — Managing Docker Images for Your Laravel Project

When you rebuild or update your project, old images can pile up. Clean them from Docker Desktop:

1. Go to **Images** tab
2. Look for images named like `my-app-laravel.test` or `laravel_app`
3. To **remove an old/unused image**: click the 🗑️ **Delete** button
4. To **rebuild a fresh image**: delete the old one, then run from terminal:

```bash
sail build --no-cache    # Sail
# or
docker compose up -d --build   # Custom
```

> 💡 You can also pull official images (like `mysql:8.0` or `nginx:alpine`) directly from the Images tab by clicking **Pull** and entering the image name.

---

### 10.8 — Adjusting Resources for Your Laravel Project

If your Laravel app feels slow or your MySQL is crashing, increase Docker's resource limits:

1. Click the **Settings (⚙️)** icon in Docker Desktop
2. Go to **Resources**

| Setting | Recommended for Laravel |
|---------|------------------------|
| **CPUs** | 2–4 cores |
| **Memory** | 4–8 GB (MySQL + PHP-FPM are memory-hungry) |
| **Swap** | 1–2 GB |
| **Disk image size** | 30–60 GB (images + volumes add up fast) |

3. Click **Apply & Restart** — Docker Desktop will restart with the new limits.

---

### 10.9 — Docker Desktop Laravel Workflow Summary

Here's the full day-to-day GUI workflow for working on your Laravel project:

```
1. Open Docker Desktop
2. Containers tab → click ▶️ on your project group  → Laravel is running
3. Open browser → http://localhost (or http://localhost:8080)
4. Make code changes in your editor (files are live-synced via volumes)
5. Need to run a migration?
      → Click your app container → Terminal tab → php artisan migrate
6. Something broken? Check logs
      → Click nginx or app container → Logs tab
7. Done for the day?
      → Containers tab → click ⏹ on your project group
```

> 💡 You don't need to keep Docker Desktop open while containers are running — they keep running in the background. Desktop is just for visibility and control.

---

> 💬 **Questions or issues?** Open an issue or reach out via [@johnboscocjt](https://github.com/johnboscocjt)  
> ⭐ If this guide helped you, consider starring the repo!
