# gnupg

Container images with [GnuPG](https://gnupg.org/), the GNU Privacy Guard: `gpg` for OpenPGP, `gpgv` for checking signatures, `gpgsm` for S/MIME and `gpgtar` for encrypted archives. They are compiled from the signed release tarball of the 2.5/2.6 branch, which upstream supports long term, against the libraries of Ubuntu or Alpine. The images are rebuilt when the GnuPG Project publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the GnuPG Project. Report problems with the image in this repository and problems with GnuPG itself [upstream](https://dev.gnupg.org/).

## Quick start

Check a download against its detached signature and a keyring file with the signer's keys:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint gpgv ghcr.io/randomcontainers/gnupg \
  --keyring ./keys.gpg file.tar.gz.sig file.tar.gz
```

`gpgv` reads binary keyrings, as `gpg --export` writes them, and looks up a keyring name without a slash in its home directory, hence the `./`. Turn keys published as ASCII armor into such a file first:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/gnupg --dearmor --output keys.gpg keys.asc
```

The entrypoint runs `gpg` under `tini` in `/work`, so file names are relative to the directory you mount. A few more commands:

```sh
# Encrypt a file with a passphrase read from pass.txt, then decrypt it to
# file.out (with --batch, gpg does not overwrite an existing file)
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/gnupg \
  --batch --pinentry-mode loopback --passphrase-file pass.txt --symmetric --output file.gpg file
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/gnupg \
  --batch --pinentry-mode loopback --passphrase-file pass.txt --output file.out --decrypt file.gpg

# Pack a directory into an encrypted archive with gpgtar
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint gpgtar \
  ghcr.io/randomcontainers/gnupg --batch \
  --gpg-args "--pinentry-mode=loopback --passphrase-file=/work/pass.txt" \
  --encrypt --symmetric --output docs.gpgtar docs

# Look up a key through the Web Key Directory of its mail domain
docker run --rm ghcr.io/randomcontainers/gnupg --locate-keys torvalds@kernel.org
```

The [GnuPG manual](https://www.gnupg.org/documentation/manuals/gnupg/) covers every program and option.

## Keyrings and the home directory

`GNUPGHOME` is `/cache/.gnupg`. gpg creates it on first use, for any UID, and it goes away with the container: keys imported or generated in one `docker run` are gone in the next, and nothing secret is written to the mounted directory. To keep a keyring between runs, point `GNUPGHOME` at a directory in the mount:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -e GNUPGHOME=/work/.gnupg ghcr.io/randomcontainers/gnupg --import key.asc
```

gpg creates a missing home directory only when its name ends in `/.gnupg`; create any other with mode 700 first. Use a directory of its own rather than the `~/.gnupg` of your host, whose sockets and keyboxd database belong to the host's GnuPG. A home directory that gpg creates gets a `common.conf` with `use-keyboxd`, so keyboxd keeps the public keys in `public-keys.d/pubring.db`; private keys are in `private-keys-v1.d/`.

gpg starts `gpg-agent` and `keyboxd`, and `dirmngr` for network lookups, in the background. They stop with the container, and their sockets are in the home directory. If you mount a volume on `/cache` to keep caches, the keyring in `/cache/.gnupg`, private keys included, is kept there too.

## Passphrases

Without a terminal, gpg cannot ask for a passphrase. In scripts, pass `--batch --pinentry-mode loopback` with `--passphrase-file`, or with `--passphrase-fd 0` and `docker run -i` to read it from standard input. With `docker run -it`, gpg asks for passphrases on the terminal through `pinentry-tty`, so interactive commands such as `--full-generate-key` work as well.

## What is in the image

- `gpg`, `gpgv`, `gpgsm`, `gpgtar`, `gpg-agent`, `dirmngr`, `dirmngr-client`, `gpgconf`, `gpg-connect-agent`, `gpg-wks-client`, `gpg-wks-server`, `gpg-mail-tube`, `gpgsplit`, `gpgparsemail`, `kbxutil` and `watchgnupg` in `/usr/local/bin`; `keyboxd`, `gpg-preset-passphrase`, `gpg-protect-tool`, `gpg-check-pattern` and `gpg-pair-tool` in `/usr/local/libexec`; `addgnupghome` and `applygnupgdefaults` in `/usr/local/sbin`.
- The distro's libgcrypt, libgpg-error, libassuan, libksba, nPth, GnuTLS (for HTTPS in dirmngr), SQLite (for keyboxd and the TOFU trust model), readline, zlib and bzip2, and on Alpine libintl.
- The distro's `pinentry-tty`.

Not included: `scdaemon`, `tpm2d` and the card tools `gpg-card` and `gpg-auth`, so smartcards, security keys such as YubiKeys, and keys held in a TPM do not work; LDAP keyservers; the manual, the man pages and the translations. The Ubuntu base image already has the distro's `gpgv` 2.4 in `/usr/bin`; `/usr/local/bin` comes first in `PATH`. `gpg --version` lists the libgcrypt version and the algorithms, and the configure summary is in `/usr/local/share/randomcontainers/gnupg/buildinfo`.

New keys use GnuPG's default algorithms, ed25519 for signing and cv25519 for encryption. With the `pqc` algorithm, `--quick-generate-key` makes a Brainpool P-384 primary key instead, with a composite Kyber-768 and Brainpool P-256 encryption subkey in the LibrePGP v5 format, which GnuPG 2.4 and most other OpenPGP implementations cannot use.

## Default or slim

GnuPG's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a GnuPG release such as `2.5.24`. `<minor>` and `<major>` are its shorter forms, `2.5` and `2`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current GnuPG version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

Upstream counts 2.5 and the coming 2.6 releases as one long-term branch. When 2.6.0 comes out, `latest`, `2` and the distro tags move to it, and `2.5` stays on the last 2.5 release built here.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. GnuPG's home directory is in `/cache` as well (see [Keyrings and the home directory](#keyrings-and-the-home-directory)). How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new GnuPG release and are rebuilt when the base image changes. The packages GnuPG needs are listed in `/usr/local/share/randomcontainers/gnupg/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/gnupg:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends git \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

Git runs `gpg` from `PATH` to sign commits and tags and to check them with `git verify-commit` and `git verify-tag`. On Alpine, start from `slim-alpine` and use `apk add --no-cache git`. The entrypoint is `["tini", "--", "gpg"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new GnuPG releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/gnupg:latest \
  --repo randomcontainers/gnupg --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/gnupg:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the GnuPG release signing keys in `keys/gnupg-release.gpg`. GnuPG publishes these keys and their fingerprints at [gnupg.org/signature_key.html](https://gnupg.org/signature_key.html), and the Dockerfiles list the fingerprints too. Releases usually carry two signatures, and the build fails if either comes from a key that is not in the keyring.

## Updates

The project checks the release tarballs listed on [gnupg.org/ftp/gcrypt/gnupg](https://gnupg.org/ftp/gcrypt/gnupg/) every 15 minutes. A release is picked up once its tarball is 48 hours old and its signature is on gnupg.org. The second release signature is usually added a day after the tarball, so most builds check both. The new version and the tarball's SHA-256 are then committed to `package.yml` and the images are rebuilt. A release signed with a key that is not in `keys/gnupg-release.gpg` fails to build until the key is added. Only the newest release is built; tags of older versions stay as they were last built.

Only 2.5 and 2.6 releases are followed. Moving the images to a later stable series needs a change to `package.yml` in this repository.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes, including those for libgcrypt and GnuTLS, reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t gnupg:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The build also runs GnuPG's test suite with `make check`.

## Licenses

GnuPG is licensed under the GNU General Public License version 3 or later. Some of its files are under the GNU Lesser General Public License version 2.1 or later, and some under your choice of LGPL version 3 or later or GPL version 2 or later. `dirmngr-client` and the certificate, CRL and OCSP code of `dirmngr` are under GPL version 2 or later. Code compiled in under other licenses:

- MIT: libdns, the DNS resolver in dirmngr.
- Spencer-86: the regular expression code from Jim Tcl, originally by Henry Spencer, which gpg uses for trust signatures.
- BSD-2-Clause-Views: Jim Tcl's UTF-8 helpers for that code.
- Unicode-DFS-2016: case mapping tables generated from the Unicode Character Database.
- FSFULLRWD: two headers with the FSF's all-permissive notice, which also covers `addgnupghome` and `applygnupgdefaults`.
- Public domain: tinycdb, the database code of the CRL cache in dirmngr.

The help texts of the Web Key Directory and mail tools in `/usr/local/share/gnupg` are under CC0-1.0. GnuPG's license files from the tarball (`COPYING`, `COPYING.GPL2`, `COPYING.LGPL21`, `COPYING.LGPL3`, `COPYING.CC0` and `COPYING.other`, which holds the texts of the other licenses) and `AUTHORS`, which ends with the all-permissive notice, are in `/usr/local/share/randomcontainers/gnupg/licenses/`. The image's license label is `GPL-3.0-or-later AND GPL-2.0-or-later AND LGPL-2.1-or-later AND (LGPL-3.0-or-later OR GPL-2.0-or-later) AND MIT AND Spencer-86 AND BSD-2-Clause-Views AND Unicode-DFS-2016 AND FSFULLRWD AND CC0-1.0 AND LicenseRef-Public-Domain`, where the `LicenseRef-` entry stands for tinycdb. The Ubuntu and Alpine packages in the image keep their own licenses.

The corresponding source for each image:

- GnuPG: every version has a GitHub release in this repository, named `v<version>`, with the exact `gnupg-<version>.tar.bz2` that was compiled and its signature. The build applies no patches. After installing, it removes the translations, `gpgscm` (the test suite's Scheme interpreter), `gpg-authcode-sign.sh` (a wrapper for Windows code signing) and the card tools `gpg-card` and `gpg-auth`, and adds three help texts from the tarball's `doc/` directory. `/usr/local/share/randomcontainers/gnupg/source` lists that release and the gnupg.org download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
