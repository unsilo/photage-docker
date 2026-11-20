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
./scripts/update-photage <version>     # pin PHOTAGE_TAG to that version and go
./scripts/update-photage --no-backup   # skip the pg_dump
```

It needs no `-f` flags and no knowledge of which overlays you run — it reads
`COMPOSE_FILE` from your `.env` like every other command in these docs.

### Back out of an upgrade

```
PHOTAGE_TAG=<the version you were on>
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

## Archives

The backups above copy photage's own storage, which only the same version of
photage and Postgres can read. An **archive** is the portable alternative: a
plain folder you can read without photage, and that any later version can
restore.

Make one from **Tools → Export** in the right-hand sidebar, which opens the
Archive page. It is written under
`PHOTAGE_EXPORT_PATH`:

```
photage-2026-11-14/
  manifest.json
  photos/2024/07/IMG_1234.jpg        the original file
  photos/2024/07/IMG_1234.jpg.json   everything photage knows about it
  photos/undated/…
  library/                           the tag tree, places, imports, settings
```

What comes back on restore: every photo with its date, location and place,
rating, description, classification data and tags; albums with their cover
photos; regions; import history; classifier thresholds and caption prompts.
Thumbnails are made again. Nothing is sent back through the AI unless you ask.

What does not: accounts (restored photos belong to the first admin, and other
users register again), and which classifiers are switched on.

Every file is checked against its recorded checksum on the way in, and photos
already in the library are skipped, so restoring twice is harmless.

An archive is as large as your originals. It is a second copy of the library,
not a replacement for backing up `.env`.

### Wipe and restore

For a major upgrade, a move to new hardware, or a database you would rather
not carry forward:

1. Export from the Archive page and wait for it to finish. Check the report
   shows no failures.
2. Stop and remove the old data:

   ```bash
   cd ~/photage
   docker compose down
   sudo rm -rf ~/photage-data/db ~/photage-data/warehouse
   mkdir -p ~/photage-data/db
   sudo mkdir -p ~/photage-data/warehouse && sudo chown 1000:1000 ~/photage-data/warehouse
   ```

   Leave `~/photage-data/export` alone. Adjust the paths if yours differ.
3. Name the archive in `.env`:

   ```
   PHOTAGE_RESTORE_FROM=photage-2026-11-14
   ```
4. `docker compose up -d`. The restore starts once the app is up;
   `docker compose logs -f photage` shows it, and the Archive page shows
   progress.

The setting can stay in `.env`: it does nothing once the library has photos.
Or skip step 3, start the app, and restore from the Archive page instead.

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
