# Contributing to DragonForge Linux

## Reporting a problem

Open an issue. Include:

- The DragonForge build you are running and the role metapackage installed
- The exact command that failed, copied rather than retyped
- The output, including any package manager error lines
- The base distribution and architecture

For package problems, include the package name and version as they appear in `dpkg -l`. A
package version and a distribution version are different things, and both are needed to
reproduce anything.

## Build and repository changes

Build and repository changes are discussed in an issue before code is written. This is not
ceremony. A change to a package's dependencies can strand every client that already has the
old version installed, and it is cheaper to find that in an issue than in a lab.

## What this project is not

DragonForge is not trying to be a general purpose distribution. It is a teaching platform.
Changes that make a classroom easier to run are in scope. Changes that make DragonForge a
better general purpose desktop usually are not.

## Licensing

Contributions are accepted under the MIT license. See [LICENSE](../LICENSE).
