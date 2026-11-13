# Verifying a DragonForge package repository

A repository can serve a healthy-looking index and fail every install. These checks separate the
two. They run against a published repository and need nothing but `curl`, `dpkg`, and a shell.

Used as the acceptance gate for DragonForge builds. Generalizes to any APT repository.

## 1. Does the suite exist

```sh
BASE=https://repo.dragonforgelinux.org
SUITE=trixie
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/dists/$SUITE/Release"
```

A 404 here means the suite name is wrong or unpublished. Everything below is meaningless until
this returns 200.

## 2. Does the index download

```sh
curl -sS "$BASE/dists/$SUITE/main/binary-amd64/Packages.gz" | gunzip > /tmp/Packages
grep -c '^Package:' /tmp/Packages
```

This count is the number of stanzas the repository claims. A count of zero is a repository with
an empty index, not a repository with no packages.

## 3. Does every stanza resolve

The check that matters. An index is a promise, and this is where you find out whether the promise
is kept.

```sh
awk '/^Package:/{p=$2} /^Filename:/{print p"\t"$2}' /tmp/Packages |
while IFS="$(printf '\t')" read -r pkg path; do
    code=$(curl -sS -o /dev/null -w '%{http_code}' "$BASE/$path")
    [ "$code" = 200 ] || printf '%s\t%s\t%s\n' "$code" "$pkg" "$path"
done
```

Silence means every promised file is present. Each line is a package that will 404 when a
student installs it.

**Read this against the same base URL a client would use.** Prefixing or stripping a path segment
changes which repository the index belongs to, and a walk against the wrong prefix reports a
broken repository with complete confidence and no error.

## 4. Does the stanza describe the file

A file can be present and still not be the thing the index promised.

```sh
curl -sS -o /tmp/pkg.deb "$BASE/$path"
sha256sum /tmp/pkg.deb
# compare against the stanza's SHA256 field
dpkg-deb -f /tmp/pkg.deb Package Version Architecture
dpkg-deb -c /tmp/pkg.deb | head
```

- The `SHA256` in the stanza must match the file.
- The `Package:` and `Version:` inside the file must match the stanza. A file named for one
  version and containing another is a stale copy, and the filename will not tell you.
- `dpkg-deb -c` output that is empty means the `.deb` has no payload. It will install cleanly
  and do nothing.

## 5. Does a maintainer script actually ship

```sh
dpkg-deb -e /tmp/pkg.deb /tmp/ctl
ls /tmp/ctl
```

A package expected to run a postinst that does not contain one is a package whose payload was
built from the wrong source directory. This is invisible to every check above.

## 6. Does a role install and remove

On a disposable machine, never on the build host:

```sh
apt-get install -y dragonforge-<role>
apt-get remove -y dragonforge-<role>
apt-get install -y dragonforge-base   # base must still be satisfied
```

If removing a role disturbs the base, the dependency graph is wrong. The failure surfaces in a
classroom otherwise, which is a worse place to find it.

## 7. Are two indexes describing the same repository

When a repository is produced by more than one process, or has been rebuilt in place, two index
files can disagree about the same set of packages. Compare them:

```sh
grep -c '^Package:' index-a      # stanza count
grep -c '^Package:' index-b
comm -12 <(grep '^Package:' index-a | sort) <(grep '^Package:' index-b | sort) | wc -l
```

Two indexes that disagree on count, on ordering, and share only a fraction of their names are two
producers, not one repository at two moments. That is a finding about the build, not the network.

## 8. Is the URL in the documentation the URL that serves

Documentation drifts faster than infrastructure. Fetch the URL the docs give and confirm it
returns an index.

```sh
grep -rhoE 'https?://[^ )"'"'"']+' docs/ | sort -u |
while read -r u; do
    printf '%s\t%s\n' "$(curl -sS -o /dev/null -w '%{http_code}' "$u")" "$u"
done
```

A documented URL that 404s is a defect in the documentation and in the install path at the same
time, because the install path is what users copy from the documentation.

## Reporting

Report what you measured, with the command that measured it. A count without its command is a
rumour about an unspecified machine. Two readers running the same check on different files get
two different answers and both are correct, which is why the file has to be named, sized, and
hashed along with the result.
