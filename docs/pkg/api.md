# Package API
## The package API for Mutiny systems.

->**`Current version: pkg0`**<-

This is extremely work in progress, don't implement anything in it yet.

This document aims to explain the API available for writing packages for
Mutiny. This should be considered the definitive document to consult
when writing packages, or when developing package managers.

The key words "must", "must not", "required", "shall", "shall not",
"should", "should not", "recommended", "may", and "optional" in this
document are to be interpreted as described in [RFC
2119](https://tools.ietf.org/html/rfc2119).

# Repositories

Package managers **must** support multiple repositories. The internal
sorting of these repositories is dictated by the
[importance](#Importance); user-facing sort, (ex. when listing
repositories installed on the system) is not defined.

The file and directory layout of a valid repository is as follows,
*italicized* items representing optional items, anything else required:

- &lt;repository name>/
    - metadata/
        - *dependencies*
        - *description*
        - importance
    - *libraries*/
        - library1.lib0
    - packages/
        - openssh/
            - openssh#1.2.3.pkg0
            - openssh#2.3.4.pkg0
            - *files*/
                - *s6*/
                    - *opensshd*/
                        - *run*
                - *patches*/
                    - *openssh-1.0.0-fix.patch*
        - package/
            - package#3.pkg0

## Repository metadata

Repositories **must** have a directory in their root named `metadata`.

### Importance

Repositories **must** have a file within `metadata`, named `importance`.
The `importance` file **must** contain a valid integer; newlines are
ignored.

### Dependencies (optional)

Repositories **may** contain a file in `metadata`, named `dependencies`.
The `dependencies` file **must** contain a single repository on each
line. Repositories listed in this file are repositories which the
package manager **must** have installed prior to the installation of the
repository depending on them. The order in which they are listed is
arbitrary.

Any repository listed will have its `libraries` directory searched when
parsing package files; the order in which they are searched is
determined by the `importance` value of the repository.

Do not add repositories to `dependencies` simply because a package in
your repository depends on something in another repository. Package
managers will deal with determining what repository needs to be
installed to satisfy a dependency through usage of the [`universe`
meta-repository](#universe).

### Description (recommended, optional)

Repositories **should** contain a file within their `metadata` directory
named `description`. This file is used for giving a description of the
repository's contents.

The first line of the `description` file **should** provide a short
summary of the repository's contents, or the objective of the
repository; keep it less than 50 characters if possible.

Any lines afterwards will be used as a longer description which can
contain any arbitrary information the repository author deems useful or
applicable.

# Packages

Repositories **must** contain a directory named `packages`; if they do
not, package managers **may** ignore them entirely, as there's not much
use to a repository with no package.

## Package specifications

Package specifications (also referred to as "specs") are strings which
describe a package.

Specifications take on multiple permutations, because they are made up
of four different parts, of which only the name is required.

Given the fully-qualified spec `package#1.0:1::repository`...

-   Package name (`package`)
-   Package version (`1.0`)
-   Package slot (`1`)
-   Package repository (`repository`)

For a package spec to be valid, it **must** match the regex
`^([A-Za-z0-9_+-]+)(#[0-9a-z\._-]+)?(:[0-9\.]+)?(::[0-9A-Za-z_-]+)?$`.

Breaking it down:

-   Package name is alphanumeric, plus `_`, `+`, and `-`
-   Package version is numeric plus `.`, `_`, `-`, and lowercase alpha
    characters. (for `r1`, etc.)
-   Slot is numeric plus `.`.
-   Repository is alphanumeric plus `_`, and `-`.

All parts of a package spec are case-sensitive.

### Disambiguation

The only strictly **required** part of a spec is the inclusion of the
package name. If any other part other than the name is omitted, it will
be disambiguated in order to determine what packages can satisfy the
specification given.

If more than one package matches a specification, the package manager
**may** prompt the user to be more specific.

## Format

`.pkg0` files and `.lib0` files are effectively just shell scripts.
Package managers **should** avoid locking the user into any particular
Bourne-style shell implementation; as such, the format of these files
should adhere to shell syntax as defined in [POSIX
2016](http://pubs.opengroup.org/onlinepubs/9699919799/ "IEEE Std 1003.1™-2008, 2016 Edition").

However, a few exceptions are made for functionality that is deemed too
useful to be given up, and isn't yet specified in POSIX...

-   Declaring arrays (`array=( "foo" "bar" "baz" )`)
-   Iterating through arrays (`for item in "${array[@]}";do ... done`)
-   Appending to arrays (`array+=( "xyzzy" )`)

# Build environment

When referring to a "build environment", this document is referring to
the literal shell environment which the shell process is running in.
This means it consists of things such as variables, functions, and
current working directory.

## Phases

Everything in this section is **required** of any package manager
implementation.

"By default" refers to a package which does not define any phases or
import any libraries which define phases.

Note the difference between "not defined" and "does nothing". Packages
**must** have each phase defined, regardless of if they have any
function; if a phase listed here is not defined by either the package
manager, or the package (or a library used by the package), the package
manager **must** error out and fail, because that is an invalid package.

"Does nothing" would mean something like `pkg_init() { true; }`. "Not
defined" would mean no definition of the function.

### `pkg_init()`

This phase is ran when a build environment is created for a package
building session. Normally nothing is done, and this is a dummy
function.

Examples of other defintions could be creating a custom `PATH` and
script wrappers to be used for build systems that are stubborn to
cooperate with cross-compilation.

### `src_fetch()`

**Only ran during installation.**

This phase's purpose is to get any sources needed to make the package
being built. By default it downloads any unretrieved files specified in
the [`DOWNLOADS`](#Downloads) variable, and then verifies the files
match the checksums specified in `DOWNLOADS`.

Usually you will not need to change this.

Examples of other definitions could include retrieval of `git` sources,
`hg`, `cvs`, etc.

### `src_unpack()`

**Only ran during installation.**

This phase's purpose is to unpack any files retrived during
`src_fetch()`. By default this means it will extract any archives
downloaded into the [`BUILD`](#Build%20Environment%20Variables)
directory, and then change into `WORK`.

Examples of other definitions could include checking out `git` sources
into `WORK`, or similar.

### `src_prepare()`

**Only ran during installation.**

This phase's purpose is to prepare the package for the real build
process; so, things which are normally done before building, like
applying patches, generating Autotools scripts, etc. are to be done
here.

By default it does nothing.

### `src_configure()`

**Only ran during installation.**

This phase's purpose is to run package configuration-related steps of
the build process. Things like `./configure`, `cmake`, or writing build
configuration files would be done here.

By default it is not defined.

### `src_compile()`

**Only ran during installation.**

This phase's purpose is to run the compilation process for the package.
Things like `make`, `ninja`, etc. would be done here.

By default it is not defined.

### `src_test()`

**Only ran during installation.**

This phase's purpose is to run tests for the package being built. Things
like `make check`, `ctest`, `./setup.py test`, etc. are done here.

By default it is not defined.

### `src_install()`

**Only ran during installation.**

This phase's purpose is to run the installation for the package; so,
commands like `make install DESTDIR="${IMAGE}"`.

Under no circumstances should anything in this phase touch something
outside the build environment.

By default it is not defined.

### `pkg_preinstall()`

**Only ran during installation.**

This phase's purpose is to run any [package
triggers](#Package%20Triggers).

### `pkg_install()`

**Only ran during installation.**

## Package Triggers

Package triggers are scripts ran on directories or files being modified
during `pkg_merge()`.

So, in order to run a script when installing a package, you'd add an
item to `${PKG_PREINSTALL_TRIGGERS[@]}`. The contents of this variable
must be in the format of:

    PKG_PREINSTALL_TRIGGERS+=( "mkinitramfs:/boot:/usr/${CHOST}/lib/modules" )

## Directories

The only requirements of the directory in which a package build is
executed is that it is read-write accessable by the package manager, and
that the work directory be entirely empty before any phases are ran.

The directories in which a build is executed are undefined. However, it
is a good idea to use something located in a good location for temporary
work. A directory in `/var/tmp` is a good idea; `/tmp`, not so much, as
build directories **should** be allowed to persist for long periods of
time.
