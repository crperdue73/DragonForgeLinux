# Architecture

This document describes how DragonForge is put together and why. It is not a build guide; it is
the reasoning a maintainer needs in order to change the build without breaking it.

## The two-layer model

DragonForge separates two things that classroom distributions usually merge: **the base** and
**the role**.

```
dragonforge-base                 the floor. Every DragonForge install has it.
    └── dragonforge-server       role metapackages. A student picks one.
    └── dragonforge-desktop
    └── dragonforge-cyber
    └── dragonforge-network
    └── dragonforge-programming
```

The base carries what every course needs: the package manager configuration, the repository
trust anchor, logging, the base configuration package. It is meant to be boring and to change
rarely. A course that installs `dragonforge-base` and nothing else has a working Debian system
that can install anything else from the DragonForge repository.

A role metapackage carries opinions. `dragonforge-cyber` pulls in the analysis tooling a
defensive security module needs; `dragonforge-network` pulls in packet analysis and
configuration tooling. Roles depend on the base, never on each other. That constraint is what
allows a course to remove a role without damaging the system underneath it.

**Test that constraint on every build.** Installing a role and then removing it must leave a
system that still boots, still updates, and still installs packages. If removing
`dragonforge-server-webstack` takes the base with it, the dependency graph is wrong and the
mistake will surface in a classroom during an exam.

## The repository

Packages are published to an APT repository with two suites:

```
testing/    built and verified. What a classroom should use.
unstable/   moving. What a build host produces before verification.
```

Each suite carries a standard Debian layout:

```
<suite>/dists/trixie/main/binary-amd64/Packages
<suite>/dists/trixie/Release
<suite>/pool/main/<first-letter>/<source-package>/<package>_<version>_<arch>.deb
```

Two properties of this layout cause more broken classrooms than anything else:

**A suite name and a distribution name are different objects.** A client sources line names a
suite. The `Packages` stub inside it names a distribution in its `Filename:` fields. If a client
asks for a suite that does not exist under `dists/`, it gets a 404 regardless of how healthy the
pool is. Only `trixie` is currently published.

**An index is a promise that a file exists.** A package listed in `Packages` must be resolvable
from `pool/` at the path the stanza gives. A stanza pointing at a file that is not there produces
a 404 at install time, not at update time, which means the repository can look healthy right up
until a student tries to install the package. Verify a suite by walking every stanza and
resolving every `Filename:` field, not by checking that `Packages` downloaded.

## The install path

The install script resolves a repository, writes a sources line, and installs a role. Three
things about it are load-bearing and easy to break:

1. **The base URL must resolve to an index that exists.** A URL prefix that returns 404 for
   `<base>/testing/Packages.gz` is a dead branch, and any fallback below it silently becomes the
   only path.
2. **The suite default must match a suite the repository publishes.** A default of `stable` when
   only `trixie` is published fails after the sources line has already been written.
3. **A fallback must never be silent.** If an install falls back to a different repository, a
   local mirror, or a private address, the user has to be told in terms of what they chose. A
   user who did not configure a mirror should never be told that "the local mirror" is being
   tried.

## Guards and why order matters

The install path verifies that package updates actually succeed before continuing. That check is
correct, and it depends on shell behaviour from the top of the script (`set -euo pipefail`) to
work at all; the pipeline it reads returns the status of the last command in the pipe, and
without that option a failure in an earlier stage is invisible.

This creates a specific hazard when repairing an install path. **Fixing the repository URL and
the suite can make the update succeed, which means the guard is no longer reached, which means
the guard's own defect stops being observable.** The break that gets fixed first is the break
that stops being tested.

Repairs belong in one change: the path, the suite, the fallback, and the guard, together, with
the guard reading the command's output rather than only its exit status. A package manager can
return success while having done nothing, and no amount of checking exit codes will reveal it.

## Package naming

Package names and file names drift independently. A `.deb` named for one version can contain a
different one, and a package published under a new name can leave the old name in the pool
pointing at a stale copy. The package's control metadata is authoritative; the filename is a
convention. Any process that trusts the filename to describe the contents is one rename away
from shipping the wrong artifact.

## Build verification

A package is not built until it is checked:

- The control metadata parses and the version is what was intended
- The data archive is non-empty (a stub `.deb` with no payload installs successfully and does
  nothing)
- Any maintainer scripts the package claims to ship are actually inside it
- Every file the index promises resolves in the pool
- A role installs on the base and removes cleanly

These are cheap checks. Each one of them has been the cause of a classroom failure at some point.
