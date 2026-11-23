# chaoge-ios APT source — working notes

`https://chaoge-ios.github.io/` is a flat APT source for jailbroken iOS, served by
GitHub Pages from this repository. The address is the source address users add in
Irisin/Sileo; renaming the repository would change it.

## How the site is built

- Nothing is compiled here. `scripts/update-repository.sh` indexes `debs/*.deb`
  with `apt-ftparchive` into `Packages`, `Packages.xz` and `Release`, then copies
  `index.html`, `CydiaIcon.png`, `sileo-featured.json`, `depictions/` and a `qr.svg`
  into `_site/` and touches `_site/.nojekyll`.
- `.github/workflows/update-repository.yml` runs that script on a push to `main`
  that touches `debs/**`, `depictions/**`, `index.html`, `CydiaIcon.png`,
  `sileo-featured.json`, `scripts/update-repository.sh` or the workflow itself
  (and on `workflow_dispatch`), then uploads `_site/` with `actions/upload-pages-artifact`
  and deploys it with `actions/deploy-pages`. A push that touches none of those
  paths publishes nothing.
- Pages must stay on **GitHub Actions** (Settings ▸ Pages). While it was on
  "Deploy from a branch", the live site was Jekyll's build of the raw branch —
  it served `index.html` but `Packages` and `Release` 404'd, because those are
  generated on the runner and never committed.
- Release fields: `Origin` = owner, `Label` = `REPOSITORY_TITLE` (`chaoge-ios`,
  set in the workflow), `Description` = `Packages from $REPOSITORY_TITLE`,
  `Codename` = repository name, `Suite: stable`. The script fails on a package
  with no `Section`, so every `.deb` needs one.
- Never commit `Packages`, `Packages.xz`, `Release`, `Release.gpg`, `InRelease`
  or `_site/`; `.gitignore` already covers them.

## Publishing a package

Only through Apelin: `apelin_package_publish` with the release builds (roothide
`iphoneos-arm64e` + rootless `iphoneos-arm64` of one version in one call), or the
Distribute… page. Never copy a `.deb` into `debs/` by hand — a new version is a
new file name, never a replaced one.

## Checking the source after a push

- Build + deploy takes about a minute; Pages may serve the previous index for up
  to ten more.
- `apelin_web_fetch https://chaoge-ios.github.io/Release` must answer 200 with
  `Label: chaoge-ios` and the `Packages` / `Packages.xz` hashes; `/Packages`
  answers 200 (empty while `debs/` is empty).
- There is no `curl`, `wget` or `python3` on the device. For binary files
  (`/Packages.xz`, `/CydiaIcon.png`) `apelin_web_fetch` refuses with the content
  type and byte size — that still proves 200, since a missing path answers
  `text/html` 404 from GitHub Pages.
- A failing run cannot have its log read from here; `api.github.com` may answer
  403 rate-limited (shared address). The repository's Actions tab has it.
