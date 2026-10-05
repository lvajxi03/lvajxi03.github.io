---
title: "Escape from the Tropics"
description: "A survival kit for KDE refugees"
date: 2026-10-05T15:05:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - Software
    - Fedora
    - fvwm3
    - RPM
categories:
    - Software
---

**KDE Plasma** is a remarkably pleasant environment for work — and for pretty much everything else — so there is very little to complain about.

However, since I have already taken the first steps toward moving completely to **FVWM3**, a little consistency seems appropriate.

It is time to start replacing some of the applications from the KDE ecosystem as well.

Not all at once.

No need to panic.

First, we need a survival kit.

# Minimal Version

The absolute minimum I would find difficult to live without is:

- another terminal → **Alacritty**, because why not,
- another file manager → **PCManFM-Qt** — I used its GTK ancestor about twenty years ago, so now the Qt version gets a chance,
- Qt appearance configuration → **qt6ct** — without it, surprises await,
- somewhat more deliberate wallpaper handling → **feh**,
- a launcher slightly more sophisticated than `dmenu` → **rofi**,
- display configuration management → **autorandr** and **arandr**.

There is one additional detail with `qt6ct`: Qt applications need to know that they are supposed to use it.

An FVWM session will therefore eventually need something along the lines of:

```bash
export QT_QPA_PLATFORMTHEME=qt6ct
```

Simply installing `qt6ct` does not magically make it take control of every Qt application.

Unfortunately.

The magic comes later.

# Standard Version

The standard version contains everything from the minimal one, plus:

- Bluetooth device management → **blueman**,
- display brightness control → **brightnessctl**,
- notifications → **dunst**,
- a PolicyKit authentication agent → **lxqt-policykit**,
- screenshots → **flameshot**,
- NetworkManager connection management → **network-manager-applet**,
- an audio mixer → **pavucontrol**,
- an X11 compositor → **picom**,
- media-player control → **playerctl**,
- automatic removable-media mounting through UDisks → **udiskie**,
- a volume-control applet → **volumeicon**,
- a second terminal, for variety → **qterminal**,
- a system tray → **stalonetray**.

At this point, an innocent little window manager starts looking suspiciously like a desktop environment.

But that is fine.

We still know exactly what it consists of.

# Full Version

Naturally, the _full_ version requires the standard version and all of its dependencies, and adds:

- yet another terminal, just to prevent boredom → **kitty**,
- another file manager → **Thunar**.

Three terminals may sound excessive.

They are not.

# RPM Packages

Since **Fedora** is now my main operating system, there is an obvious next step:

package all of this as RPMs.

Right?

Right.

The packages do not even need to contain any programs or configuration files of their own.

What we need are **metapackages** — almost empty RPM packages whose main purpose is to depend on other packages.

Instead of remembering fifteen package names, I should eventually be able to type:

```bash
sudo dnf install kde-refugee-pack-standard
```

and be done with it.

## Tools

First, install the basic RPM-building tools:

```bash
sudo dnf install rpm-build rpmdevtools rpmlint
```

Create the standard build tree:

```bash
rpmdev-setuptree
```

and generate a fresh spec template:

```bash
rpmdev-newspec my-package.spec
```

A typical spec template contains fields such as:

- `Name`,
- `Version`,
- `Release`,
- `Summary`,
- `License`,
- `URL`,
- `Source`,
- `BuildRequires`,
- `Requires`.

It also contains sections such as:

- `%description`,
- `%prep`,
- `%build`,
- `%install`,
- `%files`,
- `%changelog`.

For a package consisting entirely of dependencies, most of these are unnecessary.

There are no sources.

There is nothing to compile.

There is not even anything to install into the filesystem.

## Minimal

The `kde-refugee-pack-minimal.spec` file can therefore look like this:

```rpm
Name:           kde-refugee-pack-minimal
Version:        0.0.1
Release:        1%{?dist}
Summary:        KDE refugee pack, minimal version

License:        MIT
BuildArch:      noarch

Requires:       alacritty
Requires:       qt6ct
Requires:       pcmanfm-qt
Requires:       feh
Requires:       rofi
Requires:       autorandr
Requires:       arandr

%description
Mandatory metapackage for everyone who leaves KDE for fvwm3
(minimal version)

%files

%changelog
* Sat Oct 03 2026 Marcin Bielewicz <marcin.bielewicz@gmail.com>
- initial version
```

> Important: keep the empty `%files` section.

The package does not contain any files, but we still want `rpmbuild` to produce a binary RPM.

That is its entire purpose:

to exist and require other packages.

## Standard

The next level depends on the first one:

```rpm
Name:           kde-refugee-pack-standard
Version:        0.0.1
Release:        1%{?dist}
Summary:        KDE refugee pack, standard version

License:        MIT
BuildArch:      noarch

Requires:       kde-refugee-pack-minimal = %{version}
Requires:       blueman
Requires:       brightnessctl
Requires:       dunst
Requires:       flameshot
Requires:       lxqt-policykit
Requires:       network-manager-applet
Requires:       pavucontrol
Requires:       picom
Requires:       playerctl
Requires:       udiskie
Requires:       volumeicon
Requires:       qterminal
Requires:       stalonetray

%description
Mandatory metapackage for everyone who leaves KDE for fvwm3
(standard version)

%files

%changelog
* Sat Oct 03 2026 Marcin Bielewicz <marcin.bielewicz@gmail.com>
- initial version
```

The important line is:

```rpm
Requires:       kde-refugee-pack-minimal = %{version}
```

which means:

> I require `kde-refugee-pack-minimal` in the same version as this package.

Of course, this could be written explicitly:

```rpm
Requires:       kde-refugee-pack-minimal = 0.0.1
```

but `%{version}` saves us from having to remember to change it in the next release.

Laziness.

Once again, the driving force behind computer science.

## Full

The final level:

```rpm
Name:           kde-refugee-pack-full
Version:        0.0.1
Release:        1%{?dist}
Summary:        KDE refugee pack, full version

License:        MIT
BuildArch:      noarch

Requires:       kde-refugee-pack-standard = %{version}
Requires:       kitty
Requires:       Thunar

%description
Mandatory metapackage for everyone who leaves KDE for fvwm3
(full version)

%files

%changelog
* Sat Oct 03 2026 Marcin Bielewicz <marcin.bielewicz@gmail.com>
- initial version
```

This gives us a dependency chain:

```text
full
  └── standard
        └── minimal
```

with each level adding its own collection of tools.

# Building the Packages

A binary RPM can be built with:

```bash
rpmbuild -bb package.spec
```

The result will normally appear under:

```text
~/rpmbuild/RPMS/noarch/
```

The `-bb` option builds the binary package only.

If we also want an SRPM:

```bash
rpmbuild -ba package.spec
```

That is what I will use in CI.

If the computer is doing the work anyway, there is little reason to stop it.

# Installing the Packages

A local RPM can be installed through DNF:

```bash
sudo dnf install ./package.rpm
```

The `./` matters: it tells DNF that this is a local file rather than a package name to search for in configured repositories.

DNF then resolves and installs the dependencies.

Which was the whole point of this exercise.

# GitHub Actions

Once the spec files live in a repository, rebuilding them manually every time starts looking suspiciously like a task that could be automated.

Tasks like that are dangerous.

They tend to produce CI pipelines.

The workflow can look like this:

```yaml
name: Build RPM packages

on:
  push:
    paths:
      - "specs/**"
      - ".github/workflows/rpm.yml"
  pull_request:
    paths:
      - "specs/**"
      - ".github/workflows/rpm.yml"

permissions:
  contents: read

jobs:
  build:
    name: Build ${{ matrix.package }}
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        package:
          - kde-refugee-pack-minimal
          - kde-refugee-pack-standard
          - kde-refugee-pack-full

    container:
      image: fedora:44

    steps:
      - name: Install build tools
        run: |
          dnf install -y \
            findutils \
            rpm-build \
            rpmdevtools \
            rpmlint

      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Prepare rpmbuild tree
        run: rpmdev-setuptree

      - name: Copy spec
        run: |
          cp specs/${{ matrix.package }}.spec ~/rpmbuild/SPECS/

      - name: Lint spec
        run: |
          rpmlint specs/${{ matrix.package }}.spec

      - name: Build RPM
        run: |
          rpmbuild -ba ~/rpmbuild/SPECS/${{ matrix.package }}.spec

      - name: Show result
        run: |
          find ~/rpmbuild/RPMS ~/rpmbuild/SRPMS -type f -print

      - name: Collect build artifacts
        run: |
          mkdir -p artifacts
          cp ~/rpmbuild/RPMS/noarch/*.rpm artifacts/
          cp ~/rpmbuild/SRPMS/*.src.rpm artifacts/

      - name: Upload build bundle
        uses: actions/upload-artifact@v7
        with:
          name: bundle-${{ matrix.package }}
          path: artifacts/

      - name: Upload binary RPM directly
        uses: actions/upload-artifact@v7
        with:
          path: ~/rpmbuild/RPMS/noarch/${{ matrix.package }}-*.noarch.rpm
          archive: false

  install-test:
    name: Test package installation
    needs: build
    runs-on: ubuntu-latest

    container:
      image: fedora:44

    steps:
      - name: Download kde-refugee-pack-minimal
        uses: actions/download-artifact@v8
        with:
          name: bundle-kde-refugee-pack-minimal
          path: rpms/

      - name: Download kde-refugee-pack-standard
        uses: actions/download-artifact@v8
        with:
          name: bundle-kde-refugee-pack-standard
          path: rpms/

      - name: Download kde-refugee-pack-full
        uses: actions/download-artifact@v8
        with:
          name: bundle-kde-refugee-pack-full
          path: rpms/

      - name: Install refugee pack
        run: |
          dnf install -y \
            rpms/kde-refugee-pack-minimal-*.noarch.rpm \
            rpms/kde-refugee-pack-standard-*.noarch.rpm \
            rpms/kde-refugee-pack-full-*.noarch.rpm
```

At first glance, this line may look odd:

```yaml
runs-on: ubuntu-latest
```

We are building Fedora packages, after all.

A few lines later, however:

```yaml
container:
  image: fedora:44
```

GitHub provides the runner, while the job itself executes in a Fedora 44 environment.

There is no need for a separate Fedora runner.

The matrix:

```yaml
matrix:
  package:
    - kde-refugee-pack-minimal
    - kde-refugee-pack-standard
    - kde-refugee-pack-full
```

creates three independent builds.

Each spec file is checked with `rpmlint`, built, and stored as an artifact.

And this is where things become pleasantly convenient.

A normal artifact upload creates an archive containing the RPM and SRPM.

But `actions/upload-artifact@v7` also supports:

```yaml
archive: false
```

for a single file.

So every binary RPM is additionally available directly as an `.rpm` file.

No ZIP extraction required.

Like civilized people.

The second job downloads all three build bundles and tries to install the packages on a fresh Fedora environment.

The workflow therefore verifies more than:

> the RPM builds.

It also verifies:

> DNF can resolve the dependency chain and install all three levels.

And because the workflow itself lives at:

```text
.github/workflows/rpm.yml
```

while the path filter contains:

```yaml
paths:
  - "specs/**"
  - ".github/workflows/rpm.yml"
```

changing the workflow itself triggers another run.

The pipeline therefore tests, among other things, itself.

# Summary

A small joke of a task somehow grew into:

- a list of programs I will actually use,
- three levels of a KDE survival kit,
- three RPM metapackages,
- automated linting,
- automated builds,
- an installation test,
- artifacts available both as bundles and directly as RPM files,
- and a GitHub repository: [specfiles](https://github.com/lvajxi03/specfiles/).

All because I did not want to type a dozen package names by hand.

It was laziness, Your Honor.
