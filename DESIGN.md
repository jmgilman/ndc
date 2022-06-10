# Design Document

Nix Dev Container (NDC) is a [development container][1] aimed at containerizing
development environments using a [Nix flake-based][2] design.

## Goals

1. Allow Nix-based development flows to be available for developers who may not
   have Nix installed locally.

2. Provide rich customization options via an integrated flake library.

3. Easily integrate into existing flake-based designs.

### Non-Goals

1. NDC is not a generalized container that can be used outside Visual Studo
   Code.

2. NDC does not target non-Nix based development workflows.

3. Customizing NDC is secondary to it's primarily opioniated design.

## Design

### Container

The container will utilize a Debian-based image which is bootstrapped with the
Nix runtime during the build process. It will run as a non-root user that will
be managed with [home-manager][3]. The container will ship with a minimal
home-manager configuration for bootstrapping the initial environment.

The base image will meet the following additional requirements:

- The non-root user will have `sudo` access.
- The image will prioritize using packages from the Nix store
- The Nix store will be mountable for caching purposes

### Home-manager

Environment configuration will be provided via home-manager. The default
environment will be an opioniated design that can be optionally overriden by the
end-user (via the flake library). All non-system packages will be defined in the
home-manager configuration file.

The default home-manager environment will provide a base set of modern tools
that provide helpful functionality for developers:

- [bat](https://github.com/sharkdp/bat)
- [batman](https://github.com/eth-p/bat-extras/blob/master/doc/batman.md)
- [curl](https://curl.se/)
- [fd](https://github.com/sharkdp/fd)
- [fzf](https://github.com/junegunn/fzf)
- [gawk](https://www.gnu.org/software/gawk/)
- [gh](https://github.com/cli/cli)
- [jq](https://stedolan.github.io/jq/)
- [sed](https://www.gnu.org/software/sed/)
- [ripgrep](https://github.com/BurntSushi/ripgrep)

Additionally, the following aliases will be enabled:

| Alias  | Command              |
| ------ | -------------------- |
| `..`   | `cd ..`              |
| `...`  | `cd ../..`           |
| `cat`  | `bat --paging=never` |
| `grep` | `rg`                 |
| `ll`   | `ls -la`             |
| `man`  | `batman`             |

### Zsh

The home-manager configuration will enable [zsh][4] as the primary shell. The
[oh-my-zsh][5] plugin will be enabled. Additional oh-my-zsh plugins
(i.e., shell completions) which integrate with the base set of tools will be
enabled in the configuration.

The default prompt will be set to the [pure prompt][6].

The following additional zsh options will be configured:

| Option                     | Value  |
| -------------------------- | ------ |
| `enableAutosuggestions`    | `true` |
| `enableSyntaxHighlighting` | `true` |

## Flake Library

A core feature of NDC is the flake-based library that it ships with. This is an
optional library that can be imported into an end-user's flake to greatly
increase the functionality of NDC. The flake library is designed to reduce
developer friction by automating away common sources of toil. Additionally, the
library serves as the central point for further customizing the NDC environment.

The library serves the following core functions:

1. Provides an interface for customizing the environment via home-manager.
2. Provides *engines* for installing and configuring additional tooling.
3. Provides *plugins* for configuring common development tooling.

### Customizing the Environment

The library will allow the end-user to modify and override the default
home-manager configuration by exposing a shell hook which ingests an extended
configuration, combines it with the default one, and then reloads the
environment. The user provided configuration will take priority.

### Engines

NDC takes an opionated approach to what features a development environment
should provide. These features are henceforth referred to as *engines*. An
engine abstracts away a concept, providing a common configuration point that can
be used to configure the tool backing the engine. Thus, an engine has two
components: the concept being abstracted and the CLI tool(s) backing it. Which
tool, if any, an engine uses is controlled by the end-user.

Each engine specifies a common schema via a [Nix module][7] which defines the
interface that the end-user interacts with. The engine will perform the
necessary steps to configure the backing tool using the provided configuration.
The engine will return a set with the following contents:

| Attribute   | Description                                     |
| ----------- | ----------------------------------------------- |
| `shellHook` | A shell hook for managing the engine            |
| `deps`      | A list of dependent derivations                 |
| `env`       | A (optional) partial home-manager configuration |

The following sections define the engines provided by NDC. Each section is
prefaced with the rationale supporting the engine and is then proceeded by an
overview of the schema.

#### Git Hooks

- Maintains uniformity of code.
- Reduces the number of CI failures.
- Enforces project-based standards.
- Promotes best practices.

| Field              | Type   | Description                        |
| ------------------ | ------ | ---------------------------------- |
| `backend`          | string | The backend to use                 |
| `{stage}`          | set    | A set of commit stages to commands |
| `{stage.commands}` | list   | A list of commands to run          |

#### Task Runner

- Maintains uniformity between the local environment and CI.
- Standardizes routine maintenance tasks.
- Centrally consolidates interface to supporting tools.

| Field             | Type   | Description                |
| ----------------- | ------ | -------------------------- |
| `backend`         | string | The backend to use         |
| `{task}`          | set    | A set of tasks to commands |
| `{task.commands}` | list   | A list of commands to run  |

### Plugins

Plugins are similar to engines but differ in two primary ways:

1. Plugins do not have a standardized interface.
2. Plugins are not centrally managed.

Plugin interfaces are not standardized. Meaning, a plugin author may opt to
receive whatever input is necessary for the plugin to perform its function. The
interface between NDC and a plugin is controlled through the output. A plugin
returns the same output as an engine. NDC acts as a mediator between the
end-user and a plugin: it passes the user input and then acts on the output
generated by the plugin.

Ultimately, a plugin is nothing more than a Nix expression. This allows plugins
to be defined at-will and to be externalized for reusability. Plugins are not
centrally managed by NDC. However, plugins may be merged into the NDC repository
if they prove sufficiently useful in a wide case of scenarios.

## Alternatives Considered

### Container Image

Nix provides a [container image][9] which mimics NixOS. While this would be the
ideal candiate for this project, it was rejected for the following reasons:

1. One of the goals of NDC is to enable developers to benefit from a Nix-based
   workflow without necessarily being familiar with Nix. Compared to Debian,
   NixOS is less familiar to most developers.

2. Visual Studio Code provides its own version of NodeJS at runtime. It is not
   compiled in a way that is compatible with the Nix container image. Injecting
   a solution at runtime is problematic and extremely fragile.

[1]: https://code.visualstudio.com/docs/remote/containers
[2]: https://nixos.wiki/wiki/Flakes
[3]: https://github.com/nix-community/home-manager
[4]: https://www.zsh.org/
[5]: https://ohmyz.sh/
[6]: https://github.com/sindresorhus/pure
[7]: https://nixos.wiki/wiki/Module
[8]: https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks
[9]: https://hub.docker.com/r/nixos/nix
