# yarn-plugin-nixify

Generates a [Nix] expression to build a [Yarn] v2 project (not using
zero-install).

- Provides a build environment where a `yarn` shell alias is readily available
  — no global Yarn v1 install needed.

- A default configure-phase that runs `yarn` in your project. (Should be enough
  for plain JavaScript projects.)

- A default install-phase that creates executables for you based on `"bin"` in
  your `package.json`, making your package readily installable.

- Granular fetching of dependencies in Nix, speeding up rebuilds and
  potentially allowing downloads to be shared between projects.

- Preloading of your Yarn cache into the Nix store, speeding up local
  `nix-build`.

- **No Nix installation required** for the plugin itself, so it should be safe
  to add to your project even if some developers don't use Nix.

[nix]: https://nixos.org
[yarn]: https://yarnpkg.com

## Usage

**This plugin currently depends on Yarn v2 master.**

To get a compatible version, you currently need to install Yarn as follows:

```sh
yarn set version from sources
```

To then use the Nixify plugin:

```sh
# Install the plugin
yarn plugin import https://raw.githubusercontent.com/stephank/yarn-plugin-nixify/main/dist/yarn-plugin-nixify.js

# Run Yarn as usual
yarn

# Build your project with Nix
nix-build
```

Running `yarn` with this plugin enabled will generate two files:

- `yarn-project.nix`: This file is always overwritten, and contains a basic
  derivation for your project.

- `default.nix`: Only generated if it does not exist yet. This file is intended
  to be customized with any project-specific logic you need.

This should already build successfully! But if your project needs extra build
steps, you may have to customize `default.nix` a bit. Some examples of what's
possible:

```nix
{ pkgs ? import <nixpkgs> { } }:

let

  # Example of providing a different source.
  src = pkgs.lib.cleanSource ./.;

  project = pkgs.callPackage ./yarn-project.nix {

    # Example of selecting a specific version of Node.js.
    nodejs = pkgs.nodejs-14_x;

  } src;

in project.overrideAttrs (oldAttrs: {

  # If your top-level package.json doesn't set a name, you can set one here.
  name = "myproject";

  # Example of adding dependencies to the environment.
  # Native modules sometimes need these to build.
  buildInputs = oldAttrs.buildInputs ++ [ pkgs.python3 ];

  # Example of invoking a build step in your project.
  buildPhase = ''
    yarn build
  '';

})
```

## Settings

Some additional settings are available in `.yarnrc.yml`:

- `nixExprPath` can be set to customize the path where the Nixify plugin writes
  `yarn-project.nix`. For example, if you're also using [Niv] in your project,
  you may prefer to set this to `nix/yarn-project.nix`.

- `generateDefaultNix` can be set to `false` to disable generating a
  `default.nix`. This file is only generated if it doesn't exist yet, but this
  flag can be useful if you don't want a `default.nix` at all.

- `enableNixPreload` can be set to `false` to disable preloading Yarn cache
  into the Nix store. This preloading is intended to speed up local
  `nix-build`, because Nix will not have to download dependencies again.
  Preloading does mean another copy of dependencies on disk, even if you don't
  do local Nix builds, but the size is usually not an issue on modern disks.

[niv]: https://github.com/nmattia/niv

## Hacking

```sh
# In this directory:
yarn
yarn build-dev

# In your test project:
yarn plugin import /path/to/yarn-plugin-nixify/dist/yarn-plugin-nixify.dev.js
```

(Alternatively, add a direct reference in `.yarnrc.yml`. This will likely only
work if the Nix sandbox is disabled.)
