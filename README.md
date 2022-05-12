# npins

Simple and convenient dependency pinning in Nix

<!-- badges -->
[![License][license-shield]][license-url]
[![Contributors][contributors-shield]][contributors-url]
[![Issues][issues-shield]][issues-url]
[![PRs][pr-shield]][pr-url]
[![Tests][test-shield]][test-url]
[![Matrix][matrix-image]][matrix-url]

## About

`npins` is a simple tool for handling different types of dependencies in a Nix project. It is inspired by and comparable to [Niv](https://github.com/nmattia/niv).

### Features

- Track git branches
- Track git release tags
  - Tags must roughly follow SemVer
  - GitHub/GitLab releases are intentionally ignored
- For git repositories hosted on GitHub or GitLab, `fetchTarball` is used instead of `fetchGit`
- Track Nix channels
- Track PyPi packages

## Getting Started

`npins` can be built and installed like any other Nix based project. This section provides some information on installing `npins` and runs through the different usage scenarios.

### Installation

Ideally you will want to install `npins` through an overlay (eventually managed by `npins` itself). You can however use `nix run` to try it in a shell:

```sh
nix run -f https://github.com/andir/npins/archive/master.tar.gz
```

You could also install it to your profile using `nix-env` (not recommended):

```sh
nix-env -f https://github.com/andir/npins/archive/master.tar.gz -i
```

### Usage

```console
$ npins help
npins 0.1.0

USAGE:
    npins [OPTIONS] <SUBCOMMAND>

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]

SUBCOMMANDS:
    add           Adds a new pin entry
    help          Prints this message or the help of the given subcommand(s)
    import-niv    Try to import entries from Niv
    init          Intializes the npins directory. Running this multiple times will restore/upgrade the `default.nix`
                  and never touch your sources.json
    remove        Removes one pin entry
    show          Lists the current pin entries
    update        Updates all or the given pin to the latest version
    upgrade       Upgrade the sources.json and default.nix to the latest format version. This may occasionally break
                  Nix evaluation!
```

#### Initialization

In order to start using `npins` to track any dependencies you need to first [initialize](#npins-help) the project:

```sh
npins init
```

This will create an `npins` folder with a `default.nix` and `sources.json` within. By default, the `nixpkgs-unstable` channel will be added as pin.

```console
$ npins help init
npins-init 0.1.0
Intializes the npins directory. Running this multiple times will restore/upgrade the `default.nix` and never touch your
sources.json

USAGE:
    npins init [FLAGS] [OPTIONS]

FLAGS:
        --bare    Don't add an initial `nixpkgs` entry
    -h, --help    Prints help information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]
```

#### Migrate from Niv

You can import your pins from Niv:

```sh
npins import-niv nix/sources.json
npins update
```

In your Nix configuration, simply replace `import ./nix/sources.nix` with `import ./npins` — it should be a drop-in replacement.

Note that the import functionality is minimal and only preserves the necessary information to identify the dependency, but not the actual pinned values themselves. Therefore, migrating must always come with an update (unless you do it manually).

```console
$ npins help import-niv
npins-import-niv 0.1.0
Try to import entries from Niv

USAGE:
    npins import-niv [OPTIONS] [path]

FLAGS:
    -h, --help    Prints help information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]
    -n, --name <name>           Only import one entry from Niv

ARGS:
    <path>     [default: nix/sources.json]
```

#### Adding dependencies

Some common usage examples:

```sh
npins add channel nixos-21.11
# Remove -b to fetch the latest release
npins add git https://gitlab.com/simple-nixos-mailserver/nixos-mailserver.git -b "nixos-21.11"
npins add github ytdl-org youtube-dl
npins add github ytdl-org youtube-dl -b master # Track nightly
npins add github ytdl-org youtube-dl -b master --at c7965b9fc2cae54f244f31f5373cb81a40e822ab # We want *that* commit
npins add gitlab simple-nixos-mailserver nixos-mailserver --at v2.3.0 # We want *that* tag (note: tag, not version)
npins add pypi streamlit # Use latest version
npins add pypi streamlit --at 1.9.0 # We want *that* version
npins add pypi streamlit --upper-bound 2.0.0 # We only want 1.X
```

Depending on what kind of dependency you are adding, different arguments must be provided. You always have the option to specify a version (or hash, depending on the type) you want to pin to. Otherwise, the latest available version will be fetched for you. Not all features are present on all pin types.

```console
$ npins help add
npins-add 0.1.0
Adds a new pin entry

USAGE:
    npins add [FLAGS] [OPTIONS] <SUBCOMMAND>

FLAGS:
    -n, --dry-run    Don't actually apply the changes
    -h, --help       Prints help information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]
        --name <name>           

SUBCOMMANDS:
    channel    Track a Nix channel
    git        Track a git repository
    github     Track a GitHub repository
    gitlab     Track a GitLab repository
    help       Prints this message or the help of the given subcommand(s)
    pypi       Track a package on PyPi
```

#### Removing dependencies

```console
$ npins help remove
npins-remove 0.1.0
Removes one pin entry

USAGE:
    npins remove [OPTIONS] <name>

FLAGS:
    -h, --help    Prints help information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]

ARGS:
    <name>    
```

#### Show current entries

This will print the currently pinned dependencies in a human readable format. The machine readable `sources.json` may be accessed directly, but make sure to always check the format version (see below).

```console
$ npins help show
npins-show 0.1.0
Lists the current pin entries

USAGE:
    npins show [OPTIONS]

FLAGS:
    -h, --help    Prints help information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]
```

#### Updating dependencies

You can decide to update only selected dependencies, or all at once. For some pin types, we distinguish between "find out the latest version" and "fetch the latest version". These can be controlled with the `--full` and `--partial` flags.

```console
$ npins help update
npins-update 0.1.0
Updates all or the given pin to the latest version

USAGE:
    npins update [FLAGS] [OPTIONS] [names]...

FLAGS:
    -n, --dry-run    Print the diff, but don't write back the changes
    -f, --full       Re-fetch hashes even if the version hasn't changed. Useful to make sure the derivations are in the
                     Nix store
    -h, --help       Prints help information
    -p, --partial    Don't update versions, only re-fetch hashes

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]

ARGS:
    <names>...    Update only those pins
```

#### Upgrading the pins file

To ensure compatibility across releases, the `npins/sources.json` and `npins/default.nix` are versioned. Whenever the format changes (i.e. because new pin types are added), the version number is increased. Use `npins upgrade` to automatically apply the necessary changes to the `sources.json` and to replace the `default.nix` with one for the current version. No stability guarantees are made on the Nix side across versions.

```console
$ npins help upgrade
npins-upgrade 0.1.0
Upgrade the sources.json and default.nix to the latest format version. This may occasionally break Nix evaluation!

USAGE:
    npins upgrade [OPTIONS]

FLAGS:
    -h, --help    Prints help information

OPTIONS:
    -d, --directory <folder>    Base folder for sources.json and the boilerplate default.nix [env: NPINS_DIRECTORY=]
                                [default: npins]
```

### Packaging Example

Below is an example of what an expression might look like for packaging some `foobar` dependency which is tracked using `npins`:
```nix
let
   sources = import ./npins;
   pkgs = import sources.nixpkgs {};
in pkgs.stdenv.mkDerivation {
   # Use the name and owner of the repository as package name
   pname = sources.neovim.owner + "-" + sources.neovim.repository;

   # this will set the version of the package to the git revision
   version = sources.neovim.revision;

   # or, if you are tracking a tag you can use the name of the release as
   # defined on GitHub:
   # version = sources.neovim.release_name;
   src = sources.neovim;
}

```

## Contributing

Contributions to this project are welcome in the form of GitHub Issues or PRs. Please consider the following before creating PRs:

- This project has several commit hooks configured via `pre-commit-hooks`, make sure you have these enabled and they are passing
- This readme is templated, edit [README.md.in](./README.md.in) instead (the commit hook will take care of the rest)
- If you are planning to make any considerable changes, you should first present your plans in a GitHub issue so it can be discussed
- Simplicity and ease of use is one of the design goals, please keep this in mind when making contributons

<!-- MARKDOWN LINKS & IMAGES -->

[contributors-shield]: https://img.shields.io/github/contributors/andir/npins.svg?style=for-the-badge
[contributors-url]: https://github.com/andir/npins/graphs/contributors
[issues-shield]: https://img.shields.io/github/issues/andir/npins.svg?style=for-the-badge
[issues-url]: https://github.com/andir/npins/issues
[license-shield]: https://img.shields.io/github/license/andir/npins.svg?style=for-the-badge
[license-url]: https://github.com/andir/npins/blob/master/LICENSE
[test-shield]: https://img.shields.io/github/workflow/status/andir/npins/test/master?style=for-the-badge
[test-url]: https://github.com/andir/npins/actions
[pr-shield]: https://img.shields.io/github/issues-pr/andir/npins.svg?style=for-the-badge
[pr-url]: https://github.com/andir/npins/pulls
[matrix-image]: https://img.shields.io/matrix/npins:kack.it?label=Chat%20on%20Matrix&server_fqdn=matrix.org&style=for-the-badge
[matrix-url]: https://matrix.to/#/#npins:kack.it
