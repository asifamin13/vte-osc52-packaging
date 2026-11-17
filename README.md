# VTE OSC 52 packaging

Packaging for [VTE with OSC 52 clipboard support](https://github.com/asifamin13/vte), intended for opt-in use by VTE-based terminal applications without replacing system VTE.

## Fedora RPM build

Install Git and Mock on the host:

```sh
sudo dnf install git mock
```

Configure your account's Mock access according to Fedora's Mock setup
instructions (normally membership in the `mock` group, followed by a new login).
Mock access grants root-equivalent privileges; use a trusted build account.
The scripts do not change group membership or install host packages.
Mock installs Meson, compilers, and other build dependencies inside its chroot;
host `rpm-build` is not required.

From this repository, build with the configured default Mock root, or choose
Fedora 44 explicitly:

```sh
scripts/build-rpm
scripts/build-rpm --root fedora-44-x86_64
```

The build uses Meson with GTK 3, GTK 4, and GIR enabled, with subproject
downloads disabled. RPM build-time switches:

| Switch | Default | Effect |
| --- | --- | --- |
| `gtk4` | on | GTK 4 library and `Vte-3.91` typelib |
| `vapi` | off | Vala bindings |
| `docs` | off | gi-docgen API documentation |
| `glade` | off | GTK 3 Glade catalog and icons (no Glade build dependency) |

For GTK 3 only, with all optional bindings and documentation:

```sh
scripts/build-rpm --without gtk4 --with vapi --with docs --with glade
```

Additional Mock arguments, including `--root` and `--with`/`--without`, are
passed to both SRPM creation and binary rebuild. Artifacts and logs go into
`rpm-results/` in this repository, regardless of the calling directory.
Preserve that directory before building another variant: variants use the
same RPM name/version/release and can overwrite previous artifacts.
The RPM `%check` runs exactly `vte:test-parser`.

Install the main binary RPM (not the source or debug RPMs), then opt in:

```sh
sudo dnf install ./rpm-results/vte-osc52-0.84.1-1.osc52*.x86_64.rpm
vte-osc52-run terminator --no-dbus
```

Libraries, headers, bindings, and data are private under `/opt/vte-osc52`;
only the launcher is installed under `/usr/bin`. System VTE is not replaced,
and private library provides are excluded from RPM's automatic provides
while automatic dependency requirements remain enabled. No system ldconfig
call or global environment configuration is added.

The launcher prepends private library and typelib paths only for its command.
GTK 3 consumers use `Vte-2.91`; GTK 4 consumers use `Vte-3.91` and require the
default GTK 4 build. Terminals that do not use VTE cannot use this library.

Optional development features are not discovered system-wide. Consumers must
opt into `/opt/vte-osc52/lib/pkgconfig` via `PKG_CONFIG_PATH`, Vala bindings in
`/opt/vte-osc52/share/vala/vapi` via `valac --vapidir`, and Glade data via
`GLADE_CATALOG_SEARCH_PATH=/opt/vte-osc52/share/glade/catalogs` and
`GLADE_PIXMAP_PATH=/opt/vte-osc52/share/glade/pixmaps` as appropriate. API
documentation is installed under `/opt/vte-osc52/share/doc`.

## Ubuntu deb build

Requires Git and Podman; the host need not be Ubuntu. The build runs
`dpkg-buildpackage` inside an Ubuntu 24.04 (noble) container, so it installs
the Meson toolchain and all build dependencies there; nothing is built on the
host. Override the image with `VTE_OSC52_DEB_IMAGE` (default:
`dump:5000/library/ubuntu:24.04`; edit `scripts/build-deb` if your registry
differs). Build with:

```sh
scripts/build-deb
```

The deb uses the same private-prefix layout as the RPM (`/opt/vte-osc52`, only
the launcher in `/usr/bin`). Build-time feature switches are Debian build
profiles, mirroring the RPM `%bcond` defaults:

| Profile | Default | Effect |
| --- | --- | --- |
| `pkg.vte.gtk4` | off (built) | GTK 4 library and `Vte-3.91` typelib |
| `pkg.vte.vapi` | off | Vala bindings |
| `pkg.vte.docs` | off | gi-docgen API documentation |
| `pkg.vte.glade` | off | GTK 3 Glade catalog data |

Example, GTK 3 only with Vala and docs:

```sh
DEB_BUILD_PROFILES="pkg.vte.vapi pkg.vte.docs" scripts/build-deb
```

Artifacts go into `deb-results/`; preserve it before building another
variant. The meson test runs exactly `vte:test-parser`.

Install the binary deb, then opt in:

```sh
sudo apt install ./deb-results/vte-osc52_0.85.0-1_amd64.deb
vte-osc52-run terminator --no-dbus
```

GI consumers additionally need the system typelibs the private `Vte` typelibs
reference (for example `gir1.2-freedesktop` and `gir1.2-gtk-4.0` for the GTK 4
typelib); the deb does not pull them in automatically.

## Source

The VTE fork is included as a submodule at `subs/vte`, initially pinned to tag `0.84.1-osc52.1`. Git records the exact source commit in the parent repository.

After cloning this repository, initialize the source checkout:

```sh
git submodule update --init --recursive
```

`scripts/build-rpm` and `scripts/build-deb` require the submodule HEAD to
match the gitlink recorded in the parent repository's `HEAD`. They use
`git archive` of that exact commit, not the working tree: uncommitted
submodule changes are never packaged. Temporary source files are removed on
exit. No network source fetch is used; Mock/Podman still need repository
access to install build dependencies.

## Clipboard security

OSC 52 clipboard writes allow terminal output, including output from remote programs, to replace the local clipboard. Clipboard reads can expose local clipboard contents to those programs. The intended configuration supports writes without enabling reads.
