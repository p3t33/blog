+++
title = "Scaling NixOS with \"Import All and Enable\" Pattern"
date = 2025-09-13T10:30:05+03:00
tags = ["Linux", "NixOS", "nix"]
draft = false

[cover]
image = "cover.jpg"
alt = "NixOS configuration scaling pattern"
relative = true
+++

The default structure of a fresh NixOS installation makes a lot of sense, two files,
which are intended to be used as the bases for future changes and represent
a single machine with the bare minimum, upon which this first installation is being done.
For the sake of consistent language I will call them two high level modules.

# The problem with \"all in one configuration files\"
The initial installation creates two high level modules, configuration.nix
and hardware-configuration.nix but what happens when we want to add zfs configuration or
setup home-manger, or maybe declare neovim and emacs. The cohesion of this high level modules gets
fuzzier by every such addition. Unrelated configurations get coupled together, Understating where something
starts and ends and the ability to change it becomes a nightmare.

Logically module files should encapsulate highly cohesive logic that has a single reason to change.
This has noting to do with scale(for now), and everything to do with managing code complicity and
subsequently its maintainability. This is achieved by braking up the initial monoliths into much smaller
and highly cohesive modules divided into high and low level modules. With the high level modules mostly
acting as aggregators of low level modules, and low level modules dealing with a single
responsibility(E.g, git configurations).

After this is done finding and changing things becomes much simpler. When we want to add
git configurations, just import low level git module and removing low level kvm module becomes
as easy as deleting a single import line with having kvm.nix in it.

The initial configuration.nix file was chosen as module at the very top, serving as an entry
point as it already importing hardware-configuration.nix. Next I have created multiple
high level module files on the same level as hardware-configuration.nix, each intended for
importing the low level modules:
- configuration-services.nix: for NixOS services
- sops-configuration.nix: For all the secrets.
- disko-configuration.nix: For disk partitioning, formatting, and mount.
- home-manager-as-nixos-module.nix: home-manger configuration.

Next I started to refactor my huge configuration.nix module into much smaller low level modules and
importing them into my high level modules(E.g: This way syncthing service was refactored into
syncthing.nix and was imported into configuration-services.nix). With my configuration.nix
ending having a long list of imports.

```nix
imports = [
  ./hardware-configuration.nix
  ./configuration-services.nix
  ./sops-configuration.nix
  ./disko-config.nix
  ../../modules/nixos/bootloader/systemd-boot.nix
  ../../modules/nixos/home-manager-as-nixos-module.nix
  ../../modules/nixos/fonts.nix
  ../../modules/nixos/experimental-features.nix
  ../../modules/nixos/garbage_collection.nix
  ../../modules/nixos/system_version.nix
  ../../modules/nixos/non_free_software.nix
  ../../modules/nixos/locale.nix
  ../../modules/nixos/system_packages/development.nix
  ../../modules/nixos/system_packages/cli_utilities.nix
  ../../modules/nixos/system_packages/encryption.nix
  ../../modules/nixos/system_packages/gui.nix
  ../../modules/nixos/networking/networkmanager.nix
  ../../modules/nixos/networking/hostname.nix
  ../../modules/nixos/environment_variables.nix
  ../../modules/nixos/virtualization/docker.nix
  ../../modules/nixos/virtualization/kvm.nix
  ../../modules/nixos/users.nix
  ../../modules/nixos/desktop_environment.nix
  ../../modules/nixos/dictionaries.nix
  ../../modules/nixos/security/nitrokey.nix
  ../../modules/nixos/security/solokey2.nix
  ../../modules/nixos/defaults_for_system_build.nix
  ../../modules/nixos/graphics.nix
  ../../modules/nixos/dconf.nix
  ../../modules/nixos/networking/br0_interface.nix
];
```


# Scaling for multiple machines

This initial phase worked well for me, until I decided to install NixOS on a second
machine, and yet again came across a pattern where I had to repeat myself with the
import list of all the high level modules. This got more tedious with each machine I installed
NixOS on, After I had 5 machines adding a new module meant I had to add an import line
in 5 different places, and removing a module meant I had to delete it in 5 different places.

To address this I decided mimic the way nixpkgs were doing it. E.g, in order to enable the syncthing
service all I had to do was to enable it:
```nix
services.syncthing.enable = true;
```
With no need to import it for every host that was using it.

The implementation meant that all hosts imported all the low level modules(via the high level models)
regardless if they going to use them or not and only enable the once they need.

## Wrapping up low level modules with enable option
First step was to wrap the existing modules in an enable option that will be false
by default and will need to be explicitly enabled by each machine that was using it.

This meant that every module I have created in the past had to be wrapped inside of
an option that will enable it. Some units already had an enable option, E.g:

```nix
{ config, lib, ... }:
{
  config = lib.mkIf config.programs.bat.enable {
    programs.bat = {
      config = {
        theme = "Visual Studio Dark+";
        wrap = "character";
        terminal-width = "80";
      };
    };
  };
}
```

For others I had to create such option:
```nix
{ config, lib, pkgs, ... }:

let
  cfg = config.custom.apps.gui;
in
{
  options.custom.apps.gui.enable = lib.mkEnableOption "Enable GUI utilities and graphical system tools";

  config = lib.mkIf cfg.enable {
    environment.systemPackages = with pkgs; [
      firefox
      google-chrome
    ];
  };
}
```

## Creating default.nix files
As all of my machines needed to import all of my modules it made sense to create a shared module
that will be imported by all of the machines(and so each high level module had to import a single line).
This default.nix became a single source of truth of available low level custom modules(a single file
to add and remove low level modules).

With its initial structure just being an import list
```nix
{ config, lib, ... }:
{
  imports = [
    ../custom-global-options.nix # used both for nixos and home-manager
    ./auto_upgrade.nix
    ./dconf.nix
    ./defaults_for_system_build.nix
    ./desktop_environment.nix
    ./dictionaries.nix
    ./environment_variables.nix
    ./experimental-features.nix
    ...
  ];
}
```


With my repository structured for my high level modules(partial tree):
```text
.
├── machines
│   ├── home-assistant
│   │   ├── configuration.nix
│   │   ├── disko-configuration.nix
│   │   ├── global-options.nix
│   │   ├── hardware-configuration.nix
│   │   ├── home.nix
│   │   ├── services-configuration.nix
│   │   ├── sops-configuration.nix
│   │   └── sops-home.nix
│   └── home-desktop
│       ├── configuration.nix
│       ├── disko-configuration.nix
│       ├── hardware-configuration.nix
│       ├── home.nix
│       ├── services-configuration.nix
│       ├── sops-configuration.nix
│       └── sops-home.nix
│  
└── modules
    ├── home-manager
    │   ├── alacritty.nix
    │   ├── atuin.nix
    │   └── default.nix
    └── nixos
        ├── auto_upgrade.nix
        ├── default.nix
        ├── desktop_environment.nix
        └── services
            ├── adb.nix
            ├── bluetooth.nix
            ├── calibre-web.nix
            └── default.nix
```

I have created 3 main default.nix files.

As my goal was not to replace long import list with long enable list(at the high level modules)
I used default.nix to introduce high level options to be used as a profiles(core, desktop...), each
such option can be used as aggregator of multiple low level modules to enable all of them.


```nix
{ config, lib, ... }:

let
  g = config.custom.profiles.system;
in
{
  imports = [
    ../custom-global-options.nix # used both for nixos and home-manager
    ./auto_upgrade.nix
    ./dconf.nix
    ./defaults_for_system_build.nix
    ./desktop_environment.nix
    ./dictionaries.nix
    ./environment_variables.nix
    ./experimental-features.nix
    ...
  ];

  options.custom.profiles.system = lib.mkOption {
    type = lib.types.attrsOf (lib.types.submodule {
      options.enable = lib.mkEnableOption "Enable this system profile.";
    });
    default = {};
    description = "Enable system profiles like 'desktop', 'server', 'securityKeys', etc.";
  };

  config = lib.mkMerge [
    (lib.mkIf (g.core.enable or false) {
      custom.apps.development.enable  = true;
      custom.apps.cliUtilities.enable = true;
      custom.apps.encryption.enable   = true;
    })

    (lib.mkIf (g.desktop.enable or false) {
      custom.apps.desktopEnvironment.enable = true;
      custom.apps.gui.enable = true;
      programs.dconf.enable = true;
    })

    (lib.mkIf (g.server.enable or false) {
      virtualisation.docker.enable = true;
      custom.motd.enable = true;
    })

    (lib.mkIf (g.virtualization.enable or false) {
     virtualisation.docker.enable = true;
      virtualisation.libvirtd.enable = true;
    })

    (lib.mkIf (g.securityKeys.enable or false) {
      custom.security.nitrokey.enable = true;
      custom.security.solokey2.enable = true;
    })

    (lib.mkIf (g.gaming.enable or false) {
      programs.steam.enable = true;
    })
  ];
}

```

## Tying it all together

Each high level module such as configuration.nix(after it imported other high level modules
that included all the low level modules via the default.nix modules) could enable profiles
with a single line, or enable specific low level module that is not part of
any profile.
```nix
{ pkgs, ... }:
{
  imports = [
    ./hardware-configuration.nix
    ./services-configuration.nix
    ./sops-configuration.nix
    ./disko-configuration.nix
    ../../modules/nixos # imported via default.nix
  ];

  custom = {
    profiles.system = {
      core.enable = true;
      desktop.enable = true;
      securityKeys.enable = true;
      virtualization.enable = true;
    };

    apps.wireshark.enable = true;
  };
}

```

Full implementation of this pattern is at my [repository](https://github.com/p3t33/nixos_flake)

# A few words about custom options
As part of me doing this refactor I had to create multiple options and I think there is a value in
exploring the different "types". My guiding principle when creating an option, was to use as a narrow
scope as possible, coupling options to modules (as in most cases options extended functionality of
the modules), and only using "uncoupled" options as a last resort.

## Exposing functionality
When I was wrapping my low level modules I was doing two things, I was coupling
custom option to my module and I was exposing functionality to be used by higher level
module. At its basic form this functionality was to enable the low level module.

But This is not always the case, some times an option is used to expose functionality
to be defined later by the higher level model which will be enabling it. E.g, like the number
of threads to use for a mining service:

```nix
{ config, lib, ... }:
let
  cfg = config.custom.services.xmrig;
in
{
  options.custom.services.xmrig.numberOfThreads = lib.mkOption {
    type = lib.types.ints.between 1 256; # You can raise the max as needed
    default = 1;
    description = "Number of threads for XMRig to use (via rx array)";
  };

  config = lib.mkIf config.services.xmrig.enable {
    services.xmrig = {
      settings = {
        autosave = false;

        cpu = {
          rx = lib.genList (_: -1) cfg.numberOfThreads;
          #
        };
      };
    };
  };
}
```

Which is then set in services-configuration.nix
```nix
  custom = {
    profiles.systemServices = {
      core.enable = true;
      xmr-miner.enable = true;
    };

    services = {
      gvfs.enable = true;
      xmrig.numberOfThreads = 24;
    };
  };
```

## Sharing value
And sometimes it is a value that was set inside of one module, such as the port value of deluge:
```nix
  services.deluge = {
    web = {
      enable = true;
      openFirewall = true;
      port = 8112;
    };
  };
```

And then used inside another module such as homepage_dashboard.nix
```nix
  "deluge" = {
    description = "BitTorrent client";
    href = "http://${config.customGlobal.${hostSpecific.hostName}.ip}/deluge";
    icon = "deluge.png";
    siteMonitor = "http://${config.customGlobal.localHostIPv4}:${builtins.toString config.services.deluge.web.port}";
    statusStyle = "dot";
    widget = {
      type = "deluge";
      url = "http://${config.customGlobal.${hostSpecific.hostName}.ip}:${builtins.toString config.services.deluge.web.port}";
      password = "{{HOMEPAGE_VAR_DELUGE}}";
      enableLeechProgress = true;
    };
  };
```

Or gatus.nix
```nix
  {
    name = "deluge";
    group = media;
    url = "http://${config.customGlobal.localHostIPv4}:${toString config.services.deluge.web.port}";
    interval = "30s";
    conditions = [ "[STATUS] == 200" ];
  }

```
This way changing the value of the port at a single line, inside of deluge changes it across all the
other services that are using it, instead of updating it across multiple files.

## The uncoupled Options
There are options that should be shared across multiple files but can't be coupled to a specific
module such as specific IPv4 address:
```nix
{ lib, ...}:
{
  options.customGlobal = {
    localHostIPv4 = lib.mkOption {
      default = "127.0.0.1";
      type = lib.types.str;
      description = "Defines the static IP used by the homelab machine";
    };

    # global
    anyIPv4 = lib.mkOption {
        default = "0.0.0.0";
        type = lib.types.str;
        description = "Defines an IPv4 address that binds to all available network interfaces.";
    };
  };
}
```

Not only they are defined in a single place but when using them across multiple other modules
we won't be able to misspell them like 1270.0.0 or 0.0.0.1. Every module that will be using
localHostIPv4 or anyIPv4 values will get the one that was defined once, and if we misspell the
name of the variable the evaluation of the configurations will fail.
