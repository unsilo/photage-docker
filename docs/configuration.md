# Configuration

Everything is set in `.env`, next to `docker-compose.yml`. Compose reads that
file to expand `${...}` in the compose file — it does **not** inject anything
into the containers by itself, which is why every variable the app needs is
listed explicitly under `environment:`.

After editing `.env`:

```bash
docker compose up -d      # recreates the containers that changed
```

`docker compose restart` is **not** enough — it restarts the existing container
with its existing environment.

---

## Which stack runs

### `COMPOSE_FILE`

Colon-separated list of compose files to layer, read by Compose itself. This is
the one variable that decides *which* Photage you are running, and it is the
reason none of the commands in these docs need `-f` flags.

There are two values:

```
COMPOSE_FILE=docker-compose.yml
COMPOSE_FILE=docker-compose.yml:docker-compose.hailo.yml
```

`install.sh` writes the first. After that you rarely set this by hand:
`scripts/env-detect.sh --append` derives it from the hardware it finds — the
Hailo overlay if there is a usable card of either generation, the base file alone
if there is not. It replaces the existing line rather than adding a second, so
re-running it after a hardware change is safe.

**This variable answers one question: is there a Hailo card?** It used to answer
two: a third overlay once selected a larger image, because Moondream's Python
environment was only in that image and a Hailo-8 needed Moondream to caption at
all. The environment is built on first enable now, so every image can do
descriptions and the choice moved to the Classifiers page where it belongs. If
an old `.env` still names a captioning overlay in `COMPOSE_FILE`, remove it —
nothing publishes the tag it selects, so `pull` with it fails with
`manifest unknown`.

Rules worth knowing:

- **`docker-compose.yml` must come first.** The list is applied left to right and
  later files override earlier ones. Reversing it silently gives you the base
  image again.
- **Scalars override, lists append.** A later file replacing `image:` wins
  outright, but its `volumes:` and `devices:` entries are added to what came
  before, not substituted for it. This is why the overlays are small.
- **It is relative to the directory you run from.** Compose resolves these paths
  against the working directory, not against `.env`.
- **Every `docker compose` command in this directory picks it up**, including
  `pull`, `down` and `logs`. Run one from elsewhere and you get the base stack
  and a confusing diff.

Check what you actually assembled:

```bash
docker compose config | less
```

Missing overlay variables surface here rather than at `up` time — the Hailo
overlay uses `${VAR:?}` for the values that have no safe default, so
`docker compose config` fails loudly with the name of what is unset.

---

## Required

### `PHX_HOST`

The hostname you type in the browser. The app refuses to boot without it.

This value does more than it looks like it does. It is the endpoint's host, the
default for every image URL the UI emits, and the basis of the websocket origin
check. Set it to the name people will actually use:

- Raspberry Pi with hostname `photage` → `photage.local`
- a laptop → `localhost`
- a real DNS name → use it

Reaching the server by an address that is not this one mostly works, and then
does not: pages render, thumbnails 404 with URLs that look correct in devtools,
and LiveView never connects. See `PHOTAGE_IMAGE_URL_BASE` and
`PHOTAGE_CHECK_ORIGIN` below.

### `SECRET_KEY_BASE`

Signs and encrypts session cookies.

```bash
openssl rand -base64 48
```

Changing it logs everyone out. Losing it is not a disaster — generate a new one.

### `POSTGRES_PASSWORD`

```bash
openssl rand -hex 24
```

**Hex, not base64.** This gets interpolated into `DATABASE_URL`, and base64
output contains `/` and `+`, which change how a URL parses — Ecto either raises
`Ecto.InvalidURLError` or connects with a silently truncated password. Avoid
`@`, `:` and `#` for the same reason if you pick one by hand.

Changing it after first boot does **not** change the password in the database.
The role was created on the first run with the value that was there then.

### `PHOTAGE_ADMIN_EMAIL` / `PHOTAGE_ADMIN_PASSWORD`

Creates one confirmed admin account on first boot, and **only while the users
table is empty**. The password must be **at least 12 characters and at most 72
bytes** — the upper bound is bcrypt's, and it truncates silently past it rather
than failing, so it is enforced. The email has to contain an `@`.

There is no other way in. `/users/register` finishes over a magic-link email and
this container has no working mailer. Leaving the password blank logs a warning
and creates nothing, which is a silent lockout rather than a crash.

If you are already locked out:

```bash
docker compose exec photage \
  /app/bin/photage eval 'Photage.Release.create_admin("you@example.com", "a-long-password")'
```

Both values go inert as soon as any account exists. Change the password in the
UI after first login; there is no reason to keep the original in `.env`, though
nothing breaks if you do.

---

## Storage

### `PHOTAGE_WAREHOUSE_PATH`

Where originals and thumbnails live. Unset means the named volume
`photage-warehouse`, under `/var/lib/docker` — the SD card on a Pi.

```bash
sudo mkdir -p /mnt/photos && sudo chown 1000:1000 /mnt/photos
```

```
PHOTAGE_WAREHOUSE_PATH=/mnt/photos
```

Compose treats a value containing a slash as a bind mount and anything else as a
named volume.

The `1000:1000` ownership is not optional. The container runs as uid 1000 and
cannot write a root-owned directory; the symptom is thumbnails that never
appear, with nothing obviously wrong at startup.

Inside the container this **must** land on `/app_warehouse`. The image sets
`PHOTAGE_WAREHOUSE` to that path, and that one value is both where the ingestion
pipeline writes and where `/media` is served from. Mount the volume somewhere
else without changing it and every image 404s while the URLs stay correct.

Moving an existing library: stop the stack, copy the contents of the old
location to the new one preserving ownership (`sudo cp -a`), set the variable,
`docker compose up -d`. The database stores paths relative to the warehouse, so
nothing needs rewriting.

### `PHOTAGE_IMPORT_PATH`

The folder the import screen starts in. Defaults to `./import` next to the
compose file.

If you are writing `.env` by hand rather than using the installer, `mkdir
import` first. Docker creates a missing bind-mount source as a root-owned
directory, and the import screen then shows it empty.

### `PHOTAGE_DB_PATH`

Where Postgres keeps its data directory. Unset means the named volume
`photage-db`, which `docker compose down -v` will delete without asking.

```
PHOTAGE_DB_PATH=/mnt/data/photage/db
```

`install.sh` sets this by default, to the `db` directory under the data location
you chose. The point is narrow: a bind-mounted database survives `down -v`, and
`down -v` is the command people reach for when something is stuck.

**Do not `chown` this directory.** Unlike every other path here, it is not uid
1000's. The Postgres image starts as root, chowns its data directory to uid
**999** and re-execs; a `chown 1000` breaks it, with a permissions error that
reads like a bug in Photage. Create the directory and leave its ownership alone.

**Only on a real local filesystem.** ext4, xfs, btrfs, zfs. Postgres needs POSIX
ownership and honest `fsync`, and NFS, CIFS/SMB, exFAT and virtiofs shares give
it neither — the failure is not a clean error at startup but corruption after a
power cut. `install.sh` checks and falls back to the named volume rather than
using one. On a Pi this means the SSD, and it means the SSD directly, not a
network share mounted on it.

The mount is `/var/lib/postgresql`, the parent of `PGDATA` — Postgres 18 puts
its data at `18/docker` beneath that. Expect a `18/` subdirectory to appear.

Changing the value points the stack at a **different, empty** database; it does
not move the existing one. To move it: stop the stack, `sudo cp -a` the old
directory to the new location so ownership survives, then set the variable.

**Not a backup.** A raw data directory can only be read by the Postgres major
version that wrote it, so it does not survive an upgrade to Postgres 19 and it
is not portable to another machine. Take `pg_dump` backups as well — see
[upgrading.md](upgrading.md).

---

## Networking

### `PHOTAGE_HTTP_PORT`

Port the nginx proxy publishes. Defaults to 80.

Port 80 is the entire reason the proxy exists: it makes `http://photage.local` —
no port — a valid address, which is what the app's default image URLs and the
iOS/tvOS clients' `PHOTAGE_WEB_HOST` both assume.

Change it only if something already owns 80. If you do, **also** set
`PHOTAGE_IMAGE_URL_BASE` to include the port:

```
PHOTAGE_HTTP_PORT=8080
PHOTAGE_IMAGE_URL_BASE=http://photage.local:8080
```

Leaving the second one out gives you a working site where no image loads.

### `PHOTAGE_IMAGE_URL_BASE`

Absolute base for every `<img>` the UI emits. Defaults to `http://${PHX_HOST}`.

Set it when the proxy is not on port 80, or when something in front terminates
TLS:

```
PHOTAGE_IMAGE_URL_BASE=https://photos.example.com
```

### `PHOTAGE_CHECK_ORIGIN`

Comma-separated list of origins allowed to open a websocket. Left blank the app
derives a list from `PHX_HOST` covering the http, https and port-4000 forms,
which is right for the default setup.

Set it when you also reach the server some other way — most often by IP address:

```
PHOTAGE_CHECK_ORIGIN=http://photage.local,http://192.168.1.50
```

The failure mode is specific and easy to misread: pages render fine and then
nothing is interactive. The only evidence is a transport error in the browser
console.

`PHOTAGE_CHECK_ORIGIN=false` disables the check entirely. Only sane on a network
you fully trust, and behind a reverse proxy it is better to list the public
origin.

### `PHOTAGE_DIRECT_BIND`

Where the app container itself is published, bypassing the proxy. Defaults to
`127.0.0.1:4000` so `curl localhost:4000` works for debugging without giving the
app a second LAN origin.

`0.0.0.0:4000` exposes it properly. Be aware that two reachable origins for the
same app is how you end up with broken images for whoever used the "wrong" one.

### Putting TLS in front

There is no HTTPS in this stack. To terminate TLS elsewhere — Caddy, Traefik, a
router, Cloudflare Tunnel — point it at the proxy and set:

```
PHOTAGE_IMAGE_URL_BASE=https://photos.example.com
PHOTAGE_CHECK_ORIGIN=https://photos.example.com
```

Your terminator must forward `Upgrade` and `Connection` headers, or LiveView and
the native clients will not connect.

---

## Application

### `PHOTAGE_TAG`

Image tag to run. A version number pins it; `latest` tracks the newest
release.

Pin it. It turns upgrading into a decision rather than a side effect of
`docker compose pull`.

One image, one tag. There was once a captioning variant whose tag was derived
from this one; it is no longer published — see `COMPOSE_FILE` above.

### `PHOTAGE_ALLOW_REGISTRATION`

Self-service signup at `/users/register`. Off by default.

Two things to know before turning it on. Photage has no per-user ownership of
photos, albums or tags — a second account is a second key to the same library,
not a second library. And registration completes over a magic-link email that
this container cannot send, so `true` on its own does not produce a working
signup flow.

### `POOL_SIZE`

Ecto connection pool. Defaults to 6.

A Pi 5 has 4 cores and Postgres is competing for the same ones; the app's own
default of 10 mostly buys context switching. Raise it on a bigger machine.

### `PHOTAGE_SKIP_MIGRATIONS`

`true` starts the app without running `Photage.Release.setup()`. An escape hatch
for a bad upgrade — it gets you a shell against a database the app will not
otherwise boot on. Leave it unset in normal use; migrations are idempotent and
running them is a no-op after the first boot.

### `PHOTAGE_GEONAMES_USERNAME`

Turns a photo's GPS coordinates into country, state and place tags. **Off
unless you set this**, because it is an online lookup against geonames.org
against a quota that belongs to whoever's account is used.

```
PHOTAGE_GEONAMES_USERNAME=your-geonames-username
```

Two steps, and the second is the one everyone misses:

1. register at [geonames.org/login](https://www.geonames.org/login) — free
2. on your account page, **enable the username for the free web services**

A brand-new account can log in and still have every API call rejected until
that box is ticked. The app logs the rejection and stops asking rather than
retrying — the message names the variable.

Unset, photos import normally and keep their GPS EXIF. They get no place tags,
and can still be placed by hand. Nothing fails.

`PHOTAGE_GEONAMES_LANGUAGE` defaults to `en`. `PHOTAGE_GEONAMES_BASE_URL` is only
for the commercial endpoint (`https://secure.geonames.net`); leave it unset for
the free service.

The free tier is 20,000 credits/day and 1,000/hour per username, and a lookup
is one credit — so a large first import can exhaust an hour's allowance. That is
handled: the app waits and retries a bounded number of times, then gives up on
that photo and logs it. Re-import or re-place those later.

### `POSTGRES_USER` / `POSTGRES_DB`

Default to `photage` and `photage_prod`. Changing them after first boot renames
nothing — the volume still holds the old database, and the app will look for one
that does not exist.

---

## Hailo

Only used with `docker-compose.hailo.yml`. Run `./scripts/env-detect.sh` on
the host; it prints these filled in. Full explanation in
[hailo.md](hailo.md).

| Variable | What it is |
|---|---|
| `HAILO_GID` | numeric gid owning `/dev/hailo0`, granted to the container |
| `HAILO_PYTHON_PACKAGE` | the host's `hailo_platform` package directory |
| `HAILORT_LIB` | full path to the host's `libhailort.so.<version>` |
| `HAILORT_SONAME` | the bare soname, e.g. `libhailort.so.4.20.0` |
| `PHOTAGE_MODELS_PATH` | host directory holding your `.hef` files — required |
| `PHOTAGE_HAILO_API` | `vstreams` or `infer_model`; leave unset unless detection fails |

---

## Baked into the image

These are set in the image and are not yours to change without a rebuild:

| | |
|---|---|
| `PHOTAGE_WAREHOUSE` | `/app_warehouse` |
| `PHOTAGE_CACHE` | `/app_cache` — HEFs are looked for in `models/` under this |
| `PHOTAGE_PYTHON` | `/usr/bin/python3` (set explicitly by the compose file) |
| `PORT` | `4000` |
| dev routes | off. `/dev/dashboard` is a compile-time decision and is not in a published image. |



added 





## What the AI features give you

Photage does two separate AI jobs, and what you get depends on your hardware.
**Classification** puts tags on photos — objects, scenes, subjects.
**Descriptions** write a sentence or two about each photo.

| Your hardware | Classification | Descriptions | `COMPOSE_FILE` |
|---|---|---|---|
| No accelerator | software — slow | software — slow *(moondream)* | `docker-compose.yml` |
| **Hailo-8** (AI HAT+) | **accelerated** | software — slow *(moondream)* | `docker-compose.yml:docker-compose.hailo.yml` |
| **Hailo-10H** (AI HAT+ 2) | **accelerated** | **accelerated** *(qwen2 — ~12 s/photo)* | `docker-compose.yml:docker-compose.hailo.yml` |

Put that line in `.env` and every `docker compose` command picks it up — no `-f`
flags to remember.

`COMPOSE_FILE` only answers "do I have a Hailo card?". Which classifiers run is
decided on the Classifiers page, not here. Descriptions used to be a third
overlay pulling a larger image; the Python environment Moondream needs is
built on first enable now, so every image can caption.

Three things worth knowing before you choose:

- **A Hailo-8 cannot do descriptions.** `qwen2` needs a Hailo-10H and HailoRT
  5.3.0; there is no model, runtime version or setting that changes that. A
  Hailo-8 still accelerates classification, and `moondream` on the CPU covers
  descriptions.
- **The last row needs one extra step.** `apt install hailo-h10-all` gives you
  HailoRT 5.1.1, and no captioning model is published below 5.3.0 — so a
  stock Hailo-10H install accelerates classification but not descriptions until
  you upgrade the runtime. See
  [docs/upgrade_hailo.md](docs/upgrade_hailo.md). Classification works
  either way.
- **Everything else works with no accelerator at all** — import, albums, tags,
  ratings, smart albums, the photo viewer, the native clients. The AI is an
  addition, not a requirement, and every classifier ships disabled.

---


## What you need

| | |
|---|---|
| CPU | **arm64 only, for now** — see below |
| OS | 64-bit Linux. On a Raspberry Pi that means the **64-bit** build of Raspberry Pi OS; a 32-bit userland cannot run this image. |
| Docker | Engine 24+ with the Compose v2 plugin |
| Disk | ~3 GB for the images, plus your photo library |
| RAM | 2 GB works. 4 GB or more is comfortable. |
| Ports | 80 (change with `PHOTAGE_HTTP_PORT`) |

### The reference deployment

**A Raspberry Pi 5 booting from an NVMe drive.** That is what this is built and
tested on. Booting from NVMe puts `/var/lib/docker`, the database, the photo
library and the thumbnail cache on fast storage without any of them needing
special handling.

A Pi 4 works. A Pi 5 on an SD card works and will be slow at import, and the SD
card takes the wear from thumbnailing and Postgres.

**If you want an accelerator too, mind the PCIe budget.** A Pi 5 has **one**
PCIe lane, and the AI HAT+ and AI HAT+ 2 occupy it — as does an NVMe HAT. The
straightforward answer is an NVMe drive in a **USB 3 enclosure** alongside the
AI HAT: USB 3 is well clear of anything Photage does, and the two are on separate
buses. That is a supported and recommended configuration.

If you want both on PCIe, a 2-channel PCIe FFC adapter (a Gen 2 switch with two
downstream FFC ports) lets each HAT keep its native connector. It buys capacity
and a tidier box rather than speed. Bench-test that both devices enumerate
(`lspci -nn`, then `lspci -vv | grep -i lnksta` for 5GT/s) before any case work,
and leave `dtparam=pciex1_gen=3` off — the switch is Gen 2.

| | Storage | Accelerator |
|---|---|---|
| Baseline | NVMe boot (PCIe) | none |
| **Recommended with AI** | **NVMe over USB 3** | **Hailo-10H, AI HAT+ 2 on PCIe** |
| Tidiest with AI | NVMe HAT (PCIe, 2-ch FFC switch) | Hailo-10H on the same switch |
| Not advised | SD card | either |

### arm64 only, for now

The image is published for **`linux/arm64`** and nothing else. That covers
Raspberry Pi 4 and 5 on 64-bit Raspberry Pi OS, other arm64 boards and VMs,
Apple Silicon Macs through Docker Desktop, and arm64 NAS hardware.

It does **not** cover an x86-64 machine. `docker compose pull` there fails with
`no matching manifest for linux/amd64`, which looks like a broken publish and is
not. `linux/amd64` is a build-infrastructure problem rather than a code one —
cross-building under emulation takes hours per attempt — and will arrive when
the build moves to native runners.

Under emulation it is possible and not recommended; the numerical runtime is
exactly what QEMU is worst at:

```bash
docker run --privileged --rm tonistiigi/binfmt --install arm64
# then add to .env:  DOCKER_DEFAULT_PLATFORM=linux/arm64
```

---

## Where things live

The installer asks where your data should go and creates it. Default layout:

```
~/photage/                 the install — compose files, nginx.conf, .env
~/photage-data/
  warehouse/              originals and thumbnails — THE thing to back up
  import/                 drop photos here to import them
  models/                 AI model files (and the classifier download cache)
  db/                     the database, out of reach of `down -v`
```

Answer the installer with `/mnt/photos` and all four land there instead. Config
and data stay separate on purpose: you can delete and reinstall `~/photage`
without touching photos.

Only `photage-cache` — upload staging and downloaded models, all rebuildable —
remains a Docker named volume.

**Back up `~/photage/.env`.** It holds your credentials, and losing
`POSTGRES_PASSWORD` locks you out of your own database.

Setting the paths by hand instead? Every directory except the database must be
owned by **uid 1000** — the container runs as that user and cannot write a
directory it does not own:

```bash
sudo mkdir -p /mnt/photos/{warehouse,import,models}
sudo chown -R 1000:1000 /mnt/photos
```

Getting that wrong is quiet: thumbnails never appear, or the import folder shows
up empty. Nothing fails at startup. The database directory is the exception —
create it and leave its ownership alone, because the Postgres image manages that
itself.

---

## Everyday commands

```bash
cd ~/photage

./scripts/update-photage                # upgrade — see below
./scripts/env-detect.sh                # what can this machine do?

docker compose logs -f photage          # what it is doing
docker compose ps                      # what is running
docker compose restart photage          # restart just the app
docker compose down                    # stop; your data survives
```

With `COMPOSE_FILE` set in `.env`, none of these need `-f` flags.

**Upgrade with `./scripts/update-photage`, not a bare `docker compose pull`.** It
backs the database up first, takes the current compose files, checks your `.env`
against what the new release expects, pulls, restarts, and waits for the app to
come back. The reason it exists rather than being two commands in a doc: a
release can change `docker-compose.yml`, and Compose only passes through the
environment variables the file *on your disk* names — so a new setting can be
documented, shipped and silently doing nothing until that file is refreshed too.
`--check` shows what it would do and writes nothing. Full detail in
[docs/upgrading.md](docs/upgrading.md).

`docker compose down -v` deletes the named volumes. With the installer's
defaults your photos and database are on your own filesystem and survive it, but
do not make a habit of it.

---

## Known limitations

Listed because you will hit them, not as an apology.

- **No email.** The mailer is not wired up, so password reset and self-service
  registration do not work. The admin account is created from `.env` on first
  boot; that is the only way in.
- **One library, not one per user.** There is no per-user ownership of photos,
  albums or tags. A second account is a second key to the same library.
- **No HTTPS.** The proxy serves plain HTTP. Put Photage on a network you trust,
  or terminate TLS in front of it — see
  [docs/configuration.md](docs/configuration.md).
- **No authentication on `/media`.** Images are served before the router runs,
  so no session check applies. Anyone who can reach the server and guess a path
  can fetch an image.
- **The upload endpoint has no size limit of its own.** It is authenticated by a
  client token, but a request body is read whole into memory with no ceiling.
  The 1 GB cap in `nginx.conf` is the only thing bounding it — don't remove it,
  and don't expose the app port directly.
- **Import fails noisily on a bad path.** Typing a directory that does not exist
  into the import form crashes the page instead of reporting the error, after
  creating an empty import record.
- **Descriptions in software are slow.** `moondream` runs a 0.5B model on the
  CPU. It works; it is not quick.
- **Idle memory is higher than it should be.** The numerical runtime loads at
  boot whether or not any classifier is enabled.

---