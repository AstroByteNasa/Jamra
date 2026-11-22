# Jamra · Burning Activity Calendar

**One fair fire record from two satellites, with live early warning.**

Built by **Team Astro Byte** for the NASA Space Apps Challenge 2026 — *Harmonization of MODIS and VIIRS Hot Spots*.

### [Try Jamra →](https://astrobytenasa.github.io/Jamra/)

Jamra is an installable web app (PWA) that brings together more than 20 years of NASA FIRMS active-fire hotspot observations into one harmonized burning activity calendar, from 2003 to the present. It helps distinguish changes in burning activity from differences in satellite detection sensitivity.

![Jamra interactive map and homepage](report/before_after/final_home_desktop.jpg)

## What you can do

- **Explore:** Choose a country, a predefined region or a custom area on the interactive map.
- **Compare:** Explore the burning activity calendar, compare months and years against historical baselines, and identify unusually high or low activity.
- **Stay informed:** View recent satellite observations and early-warning levels, checked every three hours.

**Coverage:** Portugal, Spain, Greece, Türkiye, Cyprus, Lebanon, Jordan and Palestine.

## Screenshots

| Burning activity calendar | Early-warning alerts |
| :---: | :---: |
| ![Historical burning activity calendar](report/before_after/final_calendar_desktop.jpg) | ![Jamra alerts interface](report/img_team/alerts.jpg) |

<details>
<summary>Mobile experience</summary>

| Home | Calendar |
| :---: | :---: |
| <img src="report/before_after/final_home_mobile.png" alt="Jamra mobile home" width="260"> | <img src="report/before_after/final_calendar_mobile.png" alt="Jamra mobile calendar" width="260"> |

</details>

## How we harmonize MODIS and VIIRS

MODIS observes active fires at approximately 1 km resolution, while VIIRS detects hotspots at approximately 375 m resolution. Simply combining their raw counts would introduce a discontinuity because the finer-resolution sensor detects more hotspots.

Jamra uses Aqua MODIS and S-NPP VIIRS observations from overlapping years, applies quality filters, and compares six candidate approaches (M0–M5) using leave-one-year-out cross-validation. The selected model, **M3**, uses country- and month-specific scaling factors with statistical shrinkage to estimate MODIS counts on the VIIRS scale. Custom areas use local factors stabilized by the corresponding country-level estimates.

In testing on the held-out years 2022–2025, the monthly country-level mean absolute error decreased from **398.7** hotspots without harmonization to **127.1** with M3 (approximately 68% lower). NOAA-20 VIIRS observations are also converted to the S-NPP scale using an overlap-derived factor.

Jamra measures satellite-detected **hotspots, not individual fires or burned area**. Early warnings support awareness and historical comparison; they are not a substitute for official emergency alerts.

See the [model implementation](pipeline/jamra/harmonize.py) and [validation results](data/processed/model_report.json) for details.

## Data sources and credits

- [NASA FIRMS](https://earthdata.nasa.gov/firms): MODIS and VIIRS active-fire observations.
- [FIRMS historical country archives](https://firms.modaps.eosdis.nasa.gov/data/country/zips): standard-processing historical observations.
- [FIRMS API](https://firms.modaps.eosdis.nasa.gov/api/): near-real-time VIIRS observations.

We acknowledge NASA FIRMS, part of NASA's [LANCE](https://earthdata.nasa.gov/lance) and Earth Science Data and Information System (ESDIS). Maps use MapLibre GL JS (BSD-3-Clause), OpenFreeMap and © OpenStreetMap contributors. NASA does not endorse this project.

## Run locally

### Preview the web app

Install Python, then clone and serve the repository:

```bash
git clone https://github.com/AstroByteNasa/Jamra.git
cd Jamra
python -m http.server 8000 --directory web/public
```

Open [http://localhost:8000](http://localhost:8000) in your browser.

### Rebuild the data and run tests

Create a Python virtual environment:

```bash
python -m venv .venv
```

On Linux/macOS:

```bash
source .venv/bin/activate
export PYTHONPATH=pipeline
```

On Windows (PowerShell):

```powershell
.\.venv\Scripts\Activate.ps1
$env:PYTHONPATH = "pipeline"
```

Then run:

```bash
python -m pip install pyyaml pytest
python -m jamra.download
python -m jamra.build
python -m jamra.live
python -m pytest -q
python scripts/build_pages.py
python -m http.server 8000 --directory web/public
```

Historical downloads for the supported countries are approximately 150 MB. Alternatively, use `make data`, `make test`, `make live`, `make pages` and `make serve`.

For browser checks (JavaScript errors, downloads and mobile layout), install `puppeteer-core` with `npm i puppeteer-core`, then run `node scripts/site_check.mjs`.

## Architecture (always on, free)
- **History:** FIRMS yearly country archives (standard processing). `jamra.download` reads only our
  countries out of each yearly ZIP with HTTP range requests. Rebuilt monthly by `.github/workflows/archive.yml`.
- **Live:** `.github/workflows/site.yml` runs every 3 hours: FIRMS API (`VIIRS_NOAA20_NRT`, needs
  `FIRMS_MAP_KEY`) or the public FIRMS 7-day file as fallback → de-duplicated store → month-to-date vs
  past years → `live.json`, `status.json`, `alerts.json`, `alerts.xml` → GitHub Pages. If FIRMS fails,
  the last good data stay online and the status badge shows "Delayed".
- **On-demand live queries (optional):** `worker/` is a Cloudflare Worker that hides the MAP_KEY,
  caches for 15 minutes and limits box size. Put its URL in `web/public/data/site.json` → `"proxy"`.
  Without it, the app answers from the latest saved NASA snapshot.
- **Why not fetch 20 years from the API on each click?** The FIRMS API allows 5,000 transactions per
  10 minutes and each day counts; one area's full history is ~8,700 days per satellite.
- S-NPP VIIRS stopped on 2 Nov 2026 (NASA FIRMS notice), so live data come from NOAA-20, converted to
  the S-NPP scale with a factor measured on 2018–2025 (shown on the site).

## Secrets
`FIRMS_MAP_KEY` lives only in GitHub → Settings → Secrets and variables → Actions (and as a Worker
secret). Locally it is read from `../secrets.txt`, outside the repository. Never commit keys.

## Why counts and not FRP
We tested whether summed fire radiative power (FRP) is more stable between VIIRS and MODIS than hotspot counts.
It is not: the year-to-year variation of the VIIRS/MODIS ratio is 0.57 for FRP vs 0.40 for counts, and FRP was
more stable in only 1 of 5 countries (data/processed/model_report.json, frp_test). So the calendar uses counts.

## Team Astro Byte

- **Ahmed Etkaidek**: Lead developer & designer: built and launched the Jamra web app (front end, NASA FIRMS API connection, live 3-hourly updates, GitHub Pages deployment) and designed the site and brand.
- **Shahd Farraj**: Team lead: managed the team and the plan, found and prepared the NASA FIRMS data, and leads the presentation.
- **Ahmad Akhras**: Idea, debugging and presentation: helped create the Jamra idea, debugged and tested the web app as assistant developer, and co-presents the pitch.
- **Aya Salman**: Research and validation: researched the MODIS/VIIRS science and references, checked Jamra's numbers against NASA data, and prepared answers for the judges.
- **Tala Yousef**: Video and user experience: made the demo video with English subtitles, tested the site as a first-time user, and gave design feedback.

## AI disclosure

The code was developed with the assistance of an AI coding tool, directed by Ahmed Etkaidek, who designed the site, configured the NASA FIRMS API, GitHub Actions and deployment, and tested each release. The team selected the scientific method and verified the results. The demo video was made with AI tools and features an AI-generated voice-over (ElevenLabs).

## License

This project is released under the [MIT License](LICENSE).
