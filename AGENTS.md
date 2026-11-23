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
- The index files every `Package: com.chaoge.` stanza under
  `Section: Apelin&AI`, a category of this source's own: Apelin's build
  description only accepts six section values, so the rewrite happens in
  `scripts/update-repository.sh` after `apt-ftparchive` reads `debs/`, and the
  `.deb` files keep the section they were built with.
- Never commit `Packages`, `Packages.xz`, `Release`, `Release.gpg`, `InRelease`
  or `_site/`; `.gitignore` already covers them.

## Publishing a package

Only through Apelin: `apelin_package_publish` with the release builds (roothide
`iphoneos-arm64e` + rootless `iphoneos-arm64` of one version in one call), or the
Distribute… page. Never copy a `.deb` into `debs/` by hand — a new version is a
new file name, never a replaced one.

## Mirroring packages from a jailbreak backup

Some packages in `debs/` are mirrors of what a SiriPro backup holds
(`~/Documents/SiriProbackup/越狱备份/完整_<time>/01_插件依赖` and `02_用户插件`,
all `iphoneos-arm64e`), copied with the file name the backup gives them.

- **Never mirror a package `https://apt.owngoal.dev/` already carries.** The user
  installs that source right after re-jailbreaking, so a copy here is dead weight.
  Read its index first (`https://apt.owngoal.dev/Packages`, the `Package:` lines)
  and leave out every package id it lists. When one already pushed turns out to be
  there, delete the file from `debs/` and push the deletion; the next run drops its
  index entry. On 24 Nov 2026 that was twenty packages — icli, Orion, fish, grok,
  kk, iGhostVT, Alderis, Cephei, Choicy, Comet, TrollStore Lite, Inspector, Umbra
  Patcher, Apelin itself and five of its development packages.
- Leave out `com.chaoge.*` too: those are built in their own Apelin projects and
  arrive through `apelin_package_publish`.
- Push in batches. About 35 MB per push is comfortable, 60 MB works, and one 190 MB
  push failed three times with `SecureTransport error: OSStatus -36`; several small
  commits are better than one large one.

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
- `Packages` is longer than the 32 KB one `apelin_web_fetch` call returns: read it
  with `icli probe` (`NSData.dataWithContentsOfURL_`, then `initWithData:encoding:`
  on `NSString`, which the bridge hands back as a plain JS string) and count
  `Package: ` / `Section: ` with JS. A removed file downloads as `nil` there,
  which is the 404.
- A failing run cannot have its log read from here; `api.github.com` may answer
  403 rate-limited (shared address). The repository's Actions tab has it.
