# Upgrading, backing up, restoring

## Upgrading

```bash
cd ~/photage
./scripts/update-photage
```

That is the whole thing, and it is the recommended path. It backs up the
database, takes the current compose files and scripts, checks your `.env` against
what this release expects, pulls, restarts, waits for the app to report healthy,
and prints what is switched on.

```bash
./scripts/update-photage --check       # say what would change, write nothing
./scripts/update-photage 0.1.6         # pin PHOTAGE_TAG to a version and go
./scripts/update-photage --no-backup   # skip the pg_dump
```

It needs no `-f` flags and no knowledge of which overlays you run — it reads
`COMPOSE_FILE` from your `.env` like every other command in these docs.

If `scripts/update-photage` is not there, your install predates it. Take it once
by hand and it keeps itself current after that:

```bash
cd ~/photage && mkdir -p scripts
curl -fsSL https://raw.githubusercontent.com/unsilo/photage-docker/main/scripts/update-photage \
  -o scripts/update-photage && chmod +x scripts/update-photage
```

### By hand

Nothing here is wrong; it is the same steps without the checks.

```bash
cd ~/photage
docker compose pull
docker compose up -d
```

Migrations run automatically on boot and are idempotent. Volumes survive.

If `COMPOSE_FILE` is not set in `.env`, every command needs its `-f` flags
spelled out every time:

```bash
docker compose -f docker-compose.yml -f docker-compose.hailo.yml pull
docker compose -f docker-compose.yml -f docker-compose.hailo.yml up -d
```

If you pin `PHOTAGE_TAG` — and you should — `pull` fetches nothing new until you
change it. That is the point: upgrading becomes a decision rather than a side
effect.

```
PHOTAGE_TAG=0.2.0
```

### Also refresh the compose files

A release can change the compose file, `nginx.conf` or the environment it
expects. The release notes say when. To take the current versions:

```bash
cd ~/photage
for f in docker-compose.yml docker-compose.hailo.yml nginx.conf; do
  curl -fsSL "https://raw.githubusercontent.com/unsilo/photage-docker/main/$f" -o "$f"
done
docker compose up -d
```

Re-running `install.sh` does the same thing and leaves an existing `.env`
untouched.

**This is not cosmetic.** Compose only passes through environment variables that
appear in a service's `environment:` block, so a setting introduced by a release
does nothing at all until you take the compose file that mentions it —
regardless of what your `.env` says. `PHOTAGE_LOAD_GAZETTEER` is the current
example: it was documented before the compose file carried it.

### The gazetteer needs nothing

Local reverse geocoding survives upgrades untouched. The places are rows in
Postgres and the downloaded dumps live in `/app_cache/geonames`, which is the
`photage-cache` volume; neither is in the image. `PHOTAGE_LOAD_GAZETTEER=true`
stays safe to leave set, because the loader checks the table before downloading
anything.

Worth one command after any upgrade, though:

```bash
docker compose exec photage /app/bin/photage eval 'Photage.Geo.Diagnostics.print()'
```

Early versions shipped a shared geonames.org account compiled into the image.
That is gone. An install that used to get place tags with no configuration now
gets none, and the only symptom is new photos quietly landing in Unlocated — so
if this reports `mode NONE`, see [geonames.md](geonames.md).

### Back out of an upgrade

```
PHOTAGE_TAG=0.1.4
```

```bash
docker compose up -d
```

This works as long as the newer version did not run a migration the older one
cannot read. Take a database backup before a major upgrade — see below — because
a schema change is not reversible by pinning a tag.

---

## Backing up

Three things matter, and they are unequal.

| | Replaceable? |
|---|---|
| `.env` | No. Losing `SECRET_KEY_BASE` just logs everyone out, but losing `POSTGRES_PASSWORD` locks you out of your own database. |
| the warehouse (photos) | **No.** This is the actual library. |
| the database | Painfully. Albums, tags, ratings and import history are only here. |
| `photage-cache` | Yes — rebuildable. Skip it. |

### `.env`

Copy it somewhere safe. It is small and it contains your credentials, so treat
it accordingly.

### Photos

If `PHOTAGE_WAREHOUSE_PATH` points at a real directory, back that directory up
however you already back things up — rsync, Time Machine, a second disk.

If you are on the default named volume:

```bash
docker run --rm \
  -v photage_photage-warehouse:/data:ro \
  -v "$PWD":/backup \
  debian:trixie-slim \
  tar czf /backup/photage-warehouse.tar.gz -C /data .
```

This is a good argument for setting `PHOTAGE_WAREHOUSE_PATH` before you import
anything.

### Database

**`PHOTAGE_DB_PATH` is not a backup.** Putting the database on your own
filesystem protects it from `docker compose down -v`, and from nothing else. A
raw Postgres data directory can only be read by the major version that wrote
it, so it does not survive a Postgres 19 upgrade, a corrupted page, or a
deleted directory. Take dumps regardless:

```bash
cd ~/photage
docker compose exec -T db pg_dump -U photage -Fc photage_prod > photage-db-$(date +%F).dump
```

`-Fc` is the custom format — compressed, and restorable selectively. Do this
before any upgrade that changes a major version.

A cron entry, if you want one:

```cron
0 3 * * * cd /home/pi/photage && docker compose exec -T db pg_dump -U photage -Fc photage_prod > /mnt/photos/backups/photage-$(date +\%F).dump 2>/dev/null
```

---

## Restoring

### Database

```bash
cd ~/photage
docker compose up -d db
docker compose exec -T db dropdb -U photage --if-exists photage_prod
docker compose exec -T db createdb -U photage photage_prod
docker compose exec -T db pg_restore -U photage -d photage_prod < photage-db-2026-08-08.dump
docker compose up -d
```

The app runs migrations on boot, so a dump from a slightly older version is
brought forward automatically.

### Photos

Restore the warehouse directory with its ownership intact:

```bash
sudo tar xzf photage-warehouse.tar.gz -C /mnt/photos
sudo chown -R 1000:1000 /mnt/photos
```

The `chown` is not optional. The container is uid 1000, and a warehouse it
cannot write shows up as thumbnails that never appear rather than as an error.

### Onto a new machine

1. Install Docker.
2. Run `install.sh`, or copy the compose files across by hand.
3. Restore `.env` — the old one, not a freshly generated one.
4. Restore the warehouse, `chown 1000:1000`.
5. `docker compose up -d db`, restore the database dump.
6. `docker compose up -d`.

Update `PHX_HOST` if the new machine has a different name, and check
`PHOTAGE_IMAGE_URL_BASE` and `PHOTAGE_CHECK_ORIGIN` alongside it.

---

## Moving the photo library to a bigger disk

```bash
cd ~/photage
docker compose down

sudo mkdir -p /mnt/newdisk/photos
sudo cp -a /mnt/photos/. /mnt/newdisk/photos/     # -a preserves ownership
sudo chown -R 1000:1000 /mnt/newdisk/photos
```

Set `PHOTAGE_WAREHOUSE_PATH=/mnt/newdisk/photos` and:

```bash
docker compose up -d
```

The database stores paths relative to the warehouse root, so nothing needs
rewriting. Keep the old copy until you have confirmed thumbnails still load.

Moving **off** the default named volume to a real disk is the same, with one
extra step to get the data out:

```bash
docker compose down
sudo mkdir -p /mnt/photos && sudo chown 1000:1000 /mnt/photos
docker run --rm \
  -v photage_photage-warehouse:/from:ro \
  -v /mnt/photos:/to \
  debian:trixie-slim \
  sh -c 'cp -a /from/. /to/'
```

Then set `PHOTAGE_WAREHOUSE_PATH=/mnt/photos` and bring it back up.
