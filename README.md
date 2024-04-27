# poise

The Erg package manager

This package manager is bundled with erg and is available via the `erg pack` subcommand. See [here](https://github.com/erg-lang/erg/blob/main/doc/EN/tools/pack.md) for information on how to use the command.

## Requirements

* Git
* [Github CLI](https://cli.github.com/) (if you want to publish packages)

## Bootstrap

```sh
erg src/main.er -- install
```

Alternatively, you can use [ergup](https://github.com/mtshiba/ergup) to install poise automatically.

## Usage

Actually, poise is inspired by cargo (Rust's package manager) and has almost the same command options.

### Create a new package

* Creating a new package in the current directory

```sh
erg pack init
```

* Making a new directory and creating a package

```sh
erg pack new package_name
```

### Build a package

This generates the artifacts in the `build` directory.

```sh
erg pack build
```

### Check a package

This does not generate the artifacts.

```sh
erg pack check
```

### Run a package

```sh
erg pack run
```

### Test a package

This runs the test subroutines (named with `test_` prefix) in the `tests` directory.

```sh
erg pack test
```

### Publish a package

This publishes the package to [the registry](https://github.com/erg-lang/package-index).

```sh
erg pack publish
```

### Install a package

* Install the package from the current directory

```sh
erg pack install
```

* Install the package from the registry

```sh
erg pack install package_name
```

### Uninstall a package

* Uninstall the package from the current directory

```sh
erg pack uninstall
```

* Uninstall the package by specifying the name

```sh
erg pack uninstall package_name
```

### Update dependencies

```sh
erg pack update
```

### Display the package information

```sh
erg pack metadata
```

* Display the package information with json format

```sh
erg pack metadata --format json
```

### Clean the build directory

```sh
erg pack clean
```
