# Photage

Photage is a self-contained photo archiving and organising app in the true spirit of the Unsilo project. It can collect and analyze your photos, run local geolocating services and perform edge AI classifications and descrptions without any cloud involvement.

Companion apps for macOS, iOS and tvOS apps let you backup and organize your Photo Stream and cache your albums on device, letting you view them outside your home network without opening a network port.

Photage can tag images with the country, state and place name using an images geolocation info via a local copy of the Geonames database for speed and security. It uses the latest edge AI models to classify and describe your images, all local to the device; no cloud upload or account needed.

Currently the only deployment is through Docker Hub. The source code is not OSS, but could be if there is enough interest. Access to the companion apps is only through Testflight.

Contact me if you'd like to be included. 


## Quick start
### 1. Update and Install Docker
```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot

curl -sSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker
```

### 2. Install the Hailo software — optional
Only if you have a Hailo card and want to utilize it. Skip to step 3 otherwise. In that case see [docs/hailo.md](docs/hailo.md).


### 3. Install Photage
```bash
curl -fsSL https://raw.githubusercontent.com/unsilo/photage-docker/main/install.sh | bash
```

Answer the prompts: where your photos should live, and an admin email and password.

That creates a folder at `~/photage`, fetches the compose files, creates your data directories, generates secrets and places it in the `.env` file.

### 4. Geonming choices

**Place tags from GPS coordinates are on by default.** 
The first boot downloads a ~15 MB copy of the GeoNames data — no account, no quota, and no
coordinates ever leave the machine — and photos get country/state/place tags as
they import.

If you don't want geolocation information added, or want to access Geonames larger dataset
 through their API set the following values in your .env
```
PHOTAGE_LOAD_GAZETTEER=false

# optional fallback: a free geonames.org account, enabled for web services,
PHOTAGE_GEONAMES_USERNAME=your-account
PHOTAGE_GEONAMES_PASSWORD=your-password
```

Full detail, including how the two sources interact, is in
[docs/geonames.md](docs/geonames.md).


### 5. Start it

```bash
cd ~/photage
docker compose up -d
```

First start pulls 1–2 GB and then runs database migrations. Give it a few
minutes on a Pi before deciding it is stuck; `docker compose logs -f photage`
shows what it is doing.

Then open `http://<PHX_HOST>` — `http://photage.local` by default — and log in
with the admin email and password from `.env`.

Turn classifiers on at `/classifier`. They all ship disabled.


---

## Licence

Photage is free to run for personal, non-commercial use. The image is closed
source and may not be redistributed or reverse-engineered. See
[LICENSE](LICENSE).

The compose files, scripts and documentation in *this repository* are provided
so you can run and adapt the software on your own machines; the licence covers
the image, not your `.env`.
