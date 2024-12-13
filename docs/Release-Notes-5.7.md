---
layout: default
title: Release Notes - Mock 5.7 (+configs v41.3)
---

## [Release 5.7](https://rpm-software-management.github.io/mock/Release-Notes-5.7) - 2024-09-26

### New features and important changes

- Support for [hermetic builds](feature-hermetic-builds) has [been][PR#1393]
  [implemented][PR#1449].  This update introduces two new command-line options:
  `--calculate-build-deps` and `--hermetic-build`, along with the new
  `mock-hermetic-repo(1)` utility.

  Additionally, this change introduces a new [`buildroot_lock`
  plugin](Plugin-BuildrootLock), which generates a new artifact in the buildroot—a
  buildroot *lockfile*.  Users can enable this plugin explicitly by setting
  `config_opts["plugin_conf"]["buildroot_lock_enable"] = True`.

- This version addresses [issue#521][], which requested a cleanup option for
  all chroots.  A [new][PR#1337] option, `--scrub-all-chroots`, has been
  added.  It can detect leftovers in `/var/lib/mock` or `/var/cache/mock`
  and make multiple `mock --scrub=all` calls accordingly.

- Alias `dnf4` added for the `package_manager = dnf`.

  The options specific to DNF4, previously prefixed with `dnf_*`, have been
  renamed to `dnf4_*` too to avoid confusion with `dnf5_*` options.  For backward
  compatibility, the `dnf_*` prefixed variants still work, so these config pairs
  are equivalent:

  ```python
  config_opts['dnf4_install_cmd'] = 'install python3-dnf python3-dnf-plugins-core'
  config_opts['dnf_install_cmd'] = 'install python3-dnf python3-dnf-plugins-core'

  config_opts['package_manager'] = 'dnf4'
  config_opts['package_manager'] = 'dnf'
  ```

  Some of the `dnf_*` options remain unchanged because they are universal and used
  with DNF4, DNF5, or YUM, e.g., `dnf_vars`.

  While working on this rename, the rarely used `system_<PM>_command` options have
  been changed to `<PM>_system_command` to visually align with the rest of the
  package-manager-specific options. The old variants are still accepted.

- The `--addrepo` option has been updated to affect both the bootstrap chroot
  installation and the buildroot installation, as requested in [issue#1414][].
  However, be cautious, as Mock [aggressively caches the bootstrap][issue#1289].
  Always remember to run `mock -r <chroot> --scrub=bootstrap` first.
  Additionally, as more chroots are being switched to `bootstrap_image_ready =
  True`, you'll likely need to use `--addrepo` **in combination with**
  `--no-bootstrap-image`; otherwise, the bootstrap chroot installation will remain
  unaffected.

- There's a new `config_opts['bootstrap_image_skip_pull']` option that allows you
  to skip image pulling (running the `podman pull` command by Mock) when preparing
  the bootstrap chroot.  This is useful if `podman pull` is failing, for example,
  when the registry is temporarily or permanently unavailable, but the local image
  exists, or if the image reference is pointing at a local-only image.

- There's a new [ccache](Plugin-CCache) plugin option
  `config_opts['plugin_conf']['ccache_opts']['show_stats']`; if set to `True`,
  Mock prints the ccache statistics (hits/misses) to logs.

- A new option `debug` has been [added][PR#1408] to the ccache plugin. Setting
  it to `True` creates per-object debug files that are helpful when debugging
  unexpected cache misses, see [ccache docs][ccache-docs-debug].

- A new option `hashdir` has been [added][PR#1399] to the ccache plugin. Setting
  it to `False` excludes the build working directory from the hash used to
  distinguish two compilations when generating debuginfo. While this allows the
  compiler cache to be shared across different package NEVRs, it might cause the
  debuginfo to be incorrect.
  The option can be used for issue bisecting if running the debugger is
  unnecessary ([issue#1395][]).

- New Mock RPM package provides the systemd-sysusers drop-in configuration file
  for automatic `mock 135` group ID allocation.  See [rpm docs][] for more info.


### Bugfixes

- The `installed_pkgs.log` — generated by the `package_state` plugin — was
  [previously generated too early][issue#1429], after the static build
  requirements were installed but **before** the dynamic build requirements were
  resolved and installed. This led to incorrect chroot introspection for the end
  user, as the reported set of packages needed to build the given package was
  incomplete.  The new Mock version generates the `installed_pkgs.log` file after
  the dynamic build requirements are installed.


- De-duplicating bootstrap mount points for local repositories used with `--chain`
  or `--localrepo=file:///repo/on/host`.  Caused eventual problems during
  `--scrub=bootstrap`.  This bug existed in Mock <= 5.6, but after fixing the
  [issue#1414][], it got exposed by our test suite.  Related issues include
  [issue#357][] ([commit#a0a2cba3][]) and [issue#381][] ([commit#16462acc][]).

- Previously, mock.rpm created file in `/usr/share/doc/mock` directory but did
  did own this directory.

- The fuse-overlayfs package is not installed in the Fedora container images
  by default. We need to explicitly install it, otherwise running Mock inside of
  a container won't work.

- Previously, the `nspawn_args` configuration value was not applied in multiple
  internal `doChroot()` calls.  This could cause issues when custom nspawn
  arguments were needed everywhere (see [PR#1410][]).  Now, `doChroot()`
  automatically applies `nspawn_args`, shifting the responsibility from callers to
  callee.

- Several internal code locations attempt to ensure that the result directory
  exists, creating it if necessary.  However, these locations handled it
  inconsistently, sometimes neglecting to [change the ownership][issue#1467] of
  the result directory.  Now, all locations use a single method dedicated to
  result directory preparation.


### Mock Core Configs changes

- The GPG key locations for the CentOS Stream 10 Appstream debuginfo
  and Extras Common repositories were updated to point to the correct
  GPG keys.

- Anolis-7 has been EOLed at 2024-06-30. We moved the configs to `eol` directory.

- The CentOS Stream 10 configuration has been updated to use
  `quay.io/centos/centos:stream10-development` as its bootstrap image.  Since
  this image [already has the `python3-dnf-plugins-core` package
  installed](https://issues.redhat.com/browse/CS-2506), the configuration is also
  updated to set `bootstrap_image_ready = True`.  This means the image can be
  used "as is" to bootstrap the DNF stack without installing any additional
  packages into the prepared bootstrap chroot, significantly speeding up
  bootstrap preparation.

- The centos-stream-9 (and transitively centos-stream+epel-9) configuration has
  been fixed to rely on "bootstrap image" readiness for Mock builds, see
  [issue#1442][].

- OpenSuse Leap 15.4 EOLed at 07 Dec 2023 so we moved the configs to `eol` directory.

- We [updated the configuration][PR#1195] files for chroots that still use older
  RPM versions (v4.18 and earlier), which affects RHEL/CentOS 9 and older.  These
  older RPM versions were built with an incorrect default for the `%_host_cpu`
  macro in `ppc64le` chroots, where the macro incorrectly resolved to
  `powerpc64le` instead of `ppc64le`.

  This incorrect value caused issues during architecture validation, such as when
  checking `ExclusiveArch: ppc64le` for `BuildArch: noarch` packages (done
  in-chroot by `/bin/rpmbuild`).

  The incorrect macro value has now been overridden in the relevant `ppc64le`
  configuration files in Mock, ensuring that `ExcludeArch` and `ExclusiveArch`
  validations resolve correctly.

#### Following contributors contributed to this release:

- Brian J. Murrell
- Carl George
- Jakub Kadlcik
- Jiri Kyjovsky
- Julian Sikorski
- Miroslav Suchý
- Nils Philippsen
- Thomas Mendorf

Thank you!


[PR#1195]: https://github.com/rpm-software-management/mock/pull/1195
[issue#521]: https://github.com/rpm-software-management/mock/issues/521
[PR#1399]: https://github.com/rpm-software-management/mock/pull/1399
[commit#a0a2cba3]: https://github.com/rpm-software-management/mock/commit/a0a2cba3
[PR#1337]: https://github.com/rpm-software-management/mock/pull/1337
[issue#1429]: https://github.com/rpm-software-management/mock/issues/1429
[commit#16462acc]: https://github.com/rpm-software-management/mock/commit/16462acc
[issue#1442]: https://github.com/rpm-software-management/mock/issues/1442
[PR#1408]: https://github.com/rpm-software-management/mock/pull/1408
[issue#1289]: https://github.com/rpm-software-management/mock/issues/1289
[issue#1414]: https://github.com/rpm-software-management/mock/issues/1414
[issue#381]: https://github.com/rpm-software-management/mock/issues/381
[PR#1393]: https://github.com/rpm-software-management/mock/pull/1393
[issue#357]: https://github.com/rpm-software-management/mock/issues/357
[issue#1467]: https://github.com/rpm-software-management/mock/issues/1467
[issue#1395]: https://github.com/rpm-software-management/mock/issues/1395
[PR#1410]: https://github.com/rpm-software-management/mock/pull/1410
[PR#1449]: https://github.com/rpm-software-management/mock/pull/1449
[ccache-docs-debug]: https://ccache.dev/manual/4.10.html#config_debug
[rpm docs]: https://rpm-software-management.github.io/rpm/manual/users_and_groups.html