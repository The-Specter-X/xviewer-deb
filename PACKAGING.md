# Xviewer Debian packaging

This repository prepares [Linux Mint's Xviewer](https://github.com/linuxmint/xviewer)
for an eventual Debian upload. It is currently a work in progress and is **not**
an official Debian package.

The upstream project README remains in `README.md`. This file explains the
Debian packaging workflow.

## Scope

- Build machine: a dedicated, minimal Debian 13 (Trixie) virtual machine.
- Runtime test machine: a separate fresh Debian 13 installation with Cinnamon.
- Archive target: Debian unstable after sponsorship, not Debian 13.
- Current upstream release source: Xviewer 3.4.16, release tag `master.lmde7`.

A minimal build machine is the right choice. Installing build dependencies will
pull in many GTK, Cinnamon, and development packages; that is expected and
should stay confined to the build virtual machine.

## Build a test package on Debian 13

Install only the packaging tools first:

```bash
sudo apt update
sudo apt install --no-install-recommends \
  build-essential devscripts equivs git git-buildpackage \
  pristine-tar wget lintian
```

Clone this repository and import the released upstream **source** archive:

```bash
git clone https://github.com/The-Specter-X/xviewer-deb.git
cd xviewer-deb

upstream_version=3.4.16
upstream_tag=master.lmde7

wget -O ../xviewer-${upstream_version}.tar.gz \
  https://github.com/linuxmint/xviewer/archive/refs/tags/${upstream_tag}.tar.gz

mk-origtargz --package xviewer --version ${upstream_version}+ds \
  --repack --compression xz ../xviewer-${upstream_version}.tar.gz

gbp import-orig --pristine-tar --upstream-version=${upstream_version}+ds \
  ../xviewer_${upstream_version}+ds.orig.tar.xz
```

Install exactly the build dependencies declared by the package, then build
unsigned binary packages:

```bash
sudo mk-build-deps --install --remove debian/control
dpkg-buildpackage -b -us -uc
```

The generated files are in the parent directory. Check the result:

```bash
lintian --display-info --display-experimental --pedantic \
  ../xviewer_3.4.16+ds-1_amd64.changes
```

Do not use the build VM as the graphical test environment. It intentionally has
no desktop environment.

## Test on the separate Cinnamon VM

Transfer the main binary package to the fresh Cinnamon VM, then install it
using APT so that dependencies are resolved from Debian:

```bash
sudo apt update
sudo apt install ./xviewer_3.4.16+ds-1_amd64.deb
xviewer --version
```

Test opening JPEG, PNG, WebP, HEIF and SVG images; navigation; zoom; rotation;
save; fullscreen; slideshow; metadata; plugins; help; and print preview.

The `xviewer-dev` and `gir1.2-xviewer-3.0` packages are for developers and
need not be installed for normal desktop use.

## Publishing test binaries

Do not commit `.deb` files to the Git history. Create a GitHub **Release** and
attach the generated `.deb` files as release assets. Also attach:

- `.buildinfo` and `.changes`
- the `.dsc`, `.debian.tar.xz`, and `.orig.tar.xz` source files
- a `SHA256SUMS` file

Use a clearly temporary tag such as `test-3.4.16+ds-1` until the package has
been reviewed. Keeping matching source alongside binaries satisfies the
upstream GPL requirements and lets testers reproduce the build.

## Upstream releases

Use the release source archive, not Linux Mint's `packages.tar.gz` or its
pre-built `.deb` files. Those binaries are built for Linux Mint and are not
the source input for a Debian package.

A commit hash does not become obsolete: it permanently identifies one source
snapshot. For a Debian upload, that immutability is required. The release tag
is more convenient while preparing a new version, but the imported orig
tarball and its checksum are what permanently define the Debian source
package. Do not rebuild an already-versioned Debian package from a later
unreleased commit. Package a later upstream release as a new Debian version.

## Salsa and mentors

The repository already includes Salsa CI configuration. After creating your
Salsa project, push `main`, `upstream/latest`, `pristine-tar`, and tags to
it. Keep GitHub as the VCS URL until you decide that the Salsa project is the
canonical repository; then update the two `Vcs-*` fields in `debian/control`
to the exact Salsa project URL.

Before asking for sponsorship, rebuild the signed source package in a clean
Debian unstable build environment and run the tests documented in
`debian/README.source`.
