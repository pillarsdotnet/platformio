# platformio

PlatformIO packaged for Ubuntu from the upstream PyPI sdist, published to the
[Forgejo Debian registry](https://forgejo.bobbalisa.org/pillarsdotnet/-/packages)
for **noble** (24.04), **resolute** (26.04) and **stonking** (26.10).

Ubuntu's own `platformio` is 4.3.4 on noble and absent from resolute and
stonking, so there is nothing in the archive to patch — this is a fresh
packaging of the 6.2.0 release.

## What is in here, and what is not

**Only `debian/` and the CI.** No copy of the source and no orig tarball:
the sdist is fetched from PyPI at build time and checked against the sha256
in `debian/upstream/sdist.sha256`.

```
debian/
    control  rules  install  docs  copyright  changelog
    patches/relax-deps.patch, series
    upstream/sdist.sha256     # the pin: name and sha256 of the sdist
    tests/control             # autopkgtest smoke tests
    README.source             # how a version bump works
.forgejo/
    workflows/registry.yml    # the three-series matrix
    workflows/mirror.yml      # rebuild the github branch, push it to GitHub
    registry-upload           # the whole build
```

The build is:

```
PyPI API  ->  sdist URL and sha256, compared with our pin
          ->  download, sha256sum -c
          ->  unpack, rename to platformio_<ver>.orig.tar.gz, copy debian/ in
          ->  apt-get build-dep ./       (python3-ajsonrpc from the registry)
          ->  dpkg-source -b  then  -x   (this is what applies the patch)
          ->  dpkg-buildpackage -b
          ->  upload the deb to the registry
```

The source package is built and unpacked again rather than built in place
because that is what applies `debian/patches`. A plain `dpkg-buildpackage -b`
in the sdist skips the series and would quietly ship upstream's own pins.

## The one patch

`relax-deps.patch` makes one change: `bottle == 0.13.*` becomes
`bottle >= 0.12`. noble ships bottle 0.12.25, and PlatformIO's only use of
bottle is the `bottle.template()` call in
`platformio/project/integration/generator.py`, which is the same in 0.12 and
0.13.

**It does not change the binary package's dependencies.** `dh_python3`
discards PEP 440 version constraints when it turns `install_requires` into
`${python3:Depends}`; a noble build with and without the patch generated
byte-identical `Depends:` lines, `python3-bottle` unversioned in both
(measured 2026-11-05). What the patch changes is the requirement metadata the
installed package records, in `Requires-Dist` and the egg-info's
`requires.txt`. With upstream's pin, a `platformio` installed on noble
declares a requirement its own system cannot satisfy — `pip check` reports
it, and `pkg_resources.require("platformio")` raises `VersionConflict`.
PlatformIO itself never reads those pins at runtime, so this is about not
shipping a false statement rather than about fixing a failure.

Upstream's `click != 8.3` is left alone for the same reason, read the other
way: `dh_python3` discards it anyway, so rewriting it buys nothing, and
dropping it would have the package claim support for a version upstream
excludes. No series ships the 8.3.0 in question — noble has 8.1.6, resolute
8.1.8, stonking 8.3.3.

Everything else resolves from the archive as upstream declares it, on all
three series — except `ajsonrpc`, which entered Ubuntu in stonking only and
comes from
[`python-ajsonrpc`](https://forgejo.bobbalisa.org/pillarsdotnet/python-ajsonrpc)
in this registry on noble and resolute. Build that first if both are new.

## Versions

```
6.2.0-1~ppa2~ubuntu24.04.1      noble
6.2.0-1~ppa2~ubuntu26.04.1      resolute
6.2.0-1~ppa2~ubuntu26.10.1      stonking
```

`~ppa1` was built by hand in October 2026 from a packaging tree that no
longer exists; `~ppa2` is the first revision any checkout can reproduce, and
outranks it. It also outranks noble's archive 4.3.4-3 by upstream version
alone.

A rebuild that does not touch `debian/changelog` produces the version already
in the registry and gets a 409, which the script reports and does not treat
as an error.

## Bumping the version

Two edits, and they have to agree — see `debian/README.source`:

```
debian/changelog              a new entry, <ver>-1~ppa<N>, UNRELEASED
debian/upstream/sdist.sha256  sha256sum platformio-<ver>.tar.gz
```

## What is not packaged

Toolchains, platform definitions and PIO Home. PlatformIO downloads those
into `~/.platformio` on first use, per project. `pio upgrade` and the pip
self-update path hit PEP 668 on a system install, which is expected and left
alone.

## Related

- [`pioarduino`](https://forgejo.bobbalisa.org/pillarsdotnet/pioarduino) — the
  community fork for current Espressif platforms, packaged the same way. It
  ships the same `platformio` module and the same entry points, so the two
  cannot be installed together.
- [`python-ajsonrpc`](https://forgejo.bobbalisa.org/pillarsdotnet/python-ajsonrpc)
  — the backport this needs on noble and resolute.
