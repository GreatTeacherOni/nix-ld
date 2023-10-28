# nix-ld

Run unpatched dynamic binaries on NixOS.

## Where is this useful?

While many proprietary packages in nixpkgs have already been patched with
`autoPatchelfHook` patching, there are cases where patching is not possible:

- Use binary executable downloaded with third-party package managers (e.g. vscode, npm or pip) without having to patch them on every update.
- Run games or proprietary software that attempts to verify its integrity.
- Run programs that are too large for the nix store (e.g. FPGA IDEs).

While there are other solutions such as `buildFHSUserEnv` that restore a Linux file
hierarchy as found on common Linux systems (`ld-linux-x86-64.so.2`), these
sandboxes have their own weaknesses:

- setuid binaries cannot be executed inside a fhsuserenv
- inside a `buildFHSUserEnv` you can not use other sandbox tools like bwrap or 'nix build'.
- `buildFHSUserEnv` requires a subshell which does not work well with direnv

## How does nix-ld work?

Also read this [blog post](https://blog.thalheim.io/2022/12/31/nix-ld-a-clean-solution-for-issues-with-pre-compiled-executables-on-nixos/)
to get the explaination in full detail. A summary is below:

Precompiled binaries that were not created for NixOS usually have a so-called
link-loader hardcoded into them. On Linux/x86_64 this is for example
`/lib64/ld-linux-x86-64.so.2`. for glibc. NixOS, on the other hand, usually has
its dynamic linker in the glibc package in the Nix store and therefore cannot
run these binaries. Nix-ld provides a shim layer for these types of binaries. It
is installed in the same location where other Linux distributions install their
link loader, ie. `/lib64/ld-linux-x86-64.so.2` and then loads the actual link
loader as specified in the environment variable `NIX_LD`. In addition, it also
accepts a colon-separated path from library lookup paths in `NIX_LD_LIBRARY_PATH`.
This environment variable is rewritten to `LD_LIBRARY_PATH` before
passing execution to the actual ld. This allows you to specify additional
libraries that the executable needs to run.

## Installation

nix-ld is part of nixpkgs since NixOS 22.05. There one can enable it with the following
nixos setting:

```nix
{
  programs.nix-ld.enable = true;
}
```

To install `nix-ld` from the repository instead, use the following method:

```sh
$ sudo nix-channel --add https://github.com/Mic92/nix-ld/archive/main.tar.gz nix-ld
$ sudo nix-channel --update
```

`/etc/nixos/configuration.nix`

```nix
{
  imports = [
    <nix-ld/modules/nix-ld.nix>
  ];
  # The module in this repository defines a new module under (programs.nix-ld.dev) instead of (programs.nix-ld) 
  # to not collide with the nixpkgs version.
  programs.nix-ld.dev.enable = true;
}
```


### With nix flake

Add the following lines to `/etc/nixos/flake.nix`. Replace `myhostname` with the
actual hostname of your system.

```nix
# flake.nix
{
  inputs.nixpkgs.url = "github:NixOS/nixpkgs/master";
  inputs.nix-ld.url = "github:Mic92/nix-ld";
  # this line assume that you also have nixpkgs as an input
  inputs.nix-ld.inputs.nixpkgs.follows = "nixpkgs";

  outputs = { nix-ld, nixpkgs, ... }: {
    # replace `myhostname` with your actual hostname
    nixosConfigurations.myhostname = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        # ... add this line to the rest of your configuration modules
        nix-ld.nixosModules.nix-ld

        # The module in this repository defines a new module under (programs.nix-ld.dev) instead of (programs.nix-ld) 
        # to not collide with the nixpkgs version.
        { programs.nix-ld.dev.enable = true; }
      ];
    };
  };
}
```

## Usage

After setting up the nix-ld symlink as described above, one needs to set
`NIX_LD` and `NIX_LD_LIBRARY_PATH` to run executables. For example, this can
be done with a `shell.nix` in a nix-shell like this:

```nix
with import <nixpkgs> {};
mkShell {
  NIX_LD_LIBRARY_PATH = lib.makeLibraryPath [
    stdenv.cc.cc
    openssl
    # ...
  ];
  NIX_LD = lib.fileContents "${stdenv.cc}/nix-support/dynamic-linker";
}
```

A full example is shown in [`./examples/masterpdfeditor.nix`](examples/masterpdfeditor.nix).

In [nix-autobahn](https://github.com/Lassulus/nix-autobahn) there is also a
script called `nix-autobahn-ld` that automates generating shell expressions.

In [nix-alien](https://github.com/thiagokokada/nix-alien) there is another
script called `nix-alien-ld` that uses another strategy, wrapping the program in
a `writeShellScriptBin` with the `NIX_LD`/`NIX_LD_LIBRARY_PATH` environment
variables set.

To figure out what libraries a program needs, you can use `ldd` on the binary or
set the `LD_DEBUG=libs` environment variable.

<!--
## Default Configuration for nix-ld

In some scenarios, certain build systems or programs might ignore environment variables,
which could disrupt the functioning of nix-ld.

To counteract this, nix-ld implements a fallback mechanism. 
If the `NIX_LD` environment variable is not set, 
nix-ld will verify the existence of `/run/current-system/sw/share/nix-ld/lib/ld.so`. 
If this file exists, it will be used, alongside `/run/current-system/sw/share/nix-ld/lib`.

This behavior essentially defaults back to the NixOS configuration for nix-ld. 
In terms of library paths, it will default to using the paths specified in `programs.nix-ld.libraries`. 
This ensures that nix-ld can function effectively, even when its configuration 
is not explicitly defined through the `NIX_LD` environment variable.
-->

## Known Issues

### LD_LIBRARY_PATH is inherited by child processes

nix-ld is currently rewrites `NIX_LD_LIBRARY_PATH` to `LD_LIBRARY_PATH`. This
can cause problems if a program loaded with this loader executes a normal
binary, which should not get these libraries. In the future, it may be possible
to redirect execution back to nix-ld after the actual library loader has done
its job by changing the entry point in memory to fix this.

## FAQ

### How to find libraries for my executables?

You can use tools like [nix-autobahn](https://github.com/Lassulus/nix-autobahn),
[nix-alien](https://github.com/thiagokokada/nix-alien) or use
[nix-index](https://github.com/bennofs/nix-index)

### Why not set LD_LIBRARY_PATH directly instead of NIX_LD_LIBRARY_PATH?

LD_LIBRARY_PATH affects all programs, which can inject the wrong libraries in
correct build nix application that have an RPATH set in their executable.

### Does this work on non-NixOS system?

No. Normal Linux distributions will have their own link-loader. Replacing those
with nix-ld will break the system.

### My python/nodejs/ruby/$interpreter libraries do not find the libraries configured by nix-ld

Nix-ld is only used by unpatched executables that use the link loader at `/lib`
or `/lib64`. If you use for example python from nixpkgs than it will not pick
up `NIX_LD_LIBRARY_PATH` and `NIX_LD` since these types of binaries are
configured to use a glibc from the nix store. If you encounter these cases i.e.
when you are trying to use python packages installed in a virtualenv than you
need to set `LD_LIBRARY_PATH` directly. You can also create yourself a wrapper
like this:

```nix
(pkgs.writeShellScriptBin "python" ''
  export LD_LIBRARY_PATH=$NIX_LD_LIBRARY_PATH
  exec ${pkgs.python3}/bin/python "$@"
'')
```
