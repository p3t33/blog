+++
title = 'Keeping Nix Secrets with Sops: Integration and Applications'
date = 2025-04-07T18:47:34+03:00
draft = true
tags = ["sops", "sops-nix", "age", "nixos", "nix", "home-manager"]
+++

# Keeping Nix Secrets with Sops: Integration and Applications

When setting up a full NixOS instance(or a subset in the form of a stand alone home-manger module)
secrets are part of the working environment. With nix being declarative, it makes a lot of sense
to integrate them into the nix configurations, but this needs to be done in a secure manner, so even if
your configurations are put on public display your secrets will stay safe(case and point [my repository](https://github.com/p3t33/nixos_flake)
that can be used as a reference.

This integration is also the final step for a single line provisioning of a new system(using a tool such as
[nixos-anywhere](https://github.com/nix-community/nixos-anywhere)), that once booted will require no additional work to be made fully operational.

This post will cover the integration of [sops-nix](https://github.com/Mic92/sops-nix) into nixos/home-manger and its various applications
based on my use cases. With the main goal to get things up and running from start to finish(by going over all the
moving parts and gluing all of them together) and not to repeat sops-nix readme(leaving bunch of gaps for you
to fill by yourself) or to go over a single encapsulated interesting application. Even if you already have
sops-nix running I hope my application will give you a new ideas for use in your configurations.

<u>As I am using a flake this post will reflect that</u>

## Anatomy Of Sops With and nix
There are a few moving parts here that come together to allow the use of sops
with nix via sops-nix module.

- age: An encryption tool that provides a private–public key pair for encrypting
and decrypting files securely and simply.
- sops:  A secrets management tool that uses age (among other options like GPG, KMS, etc.)
to encrypt and decrypt structured configuration files, such as YAML, JSON, and .env files.
- sops-nix: A NixOS/Home Manager module that integrates sops into your system configuration.
It securely decrypts and makes SOPS-managed secrets available at runtime for use in services, environment variables, or other system paths

From this decryption it is clear that including sops-nix into your flake input by itself is pointless
and for it to work, you will need to do more things, such as create age key pair and, using sops to
create structured files with secrets.

### Creating age key pair
The first step is to install age and sops

```nix
{ pkgs, ... }:
{
  environment.systemPackages = with pkgs; [
    sops
    age
  ];
}

```

And in the case of stand alone home-manger module

```nix
{ pkgs, ... }:
{
  home.packages = with pkgs; [
    age
    sops
  ];

}

```
And then to rebuild your system.

Generate age private key:

```bash
age-keygen -o ~/.config/sops/age/keys.txt
```
You should see in the terminal the public key value, copy it as you will need it
later(E,g: Public key: age1t22ng09gsagym5fh5helnrs7t2lgfqdmx84ww009ndwwnzz8j3vq4k723r).
You can always derive it from the private key using:

```bash
age-keygen -y ~/.config/sops/age/keys.txt
```
Note, do not password protect your private key as it is used for automated
setup, at boot time.

### Creating sops configuration file
sops as cli tool operates on secret files, it looks for private keys is expected paths,
such as ~/.config/sops/age/keys.txt. It can be used with bunch of glads or it can be used with a
configuration file that will make life easier.

There are 3 main things to understand here:
1. sops configuration file is not the actual secret, it is just a convenience to avoid typing and is not required by sops-nix.
2. sops will look for the public key you state in your sops configuration file in ~/.config/sops/age/keys.txt.
3. sops automatically searches upward in the directory tree for a file named .sops.yaml, which is the name of the
configuration file.

At the root of my repository I will create .sops.yaml so the partial
directory structure will be:

```text
├── .sops.yaml
├── flake.lock
├── flake.nix
├── machines
│   ├── generic_linux_distro
│   │   ├── secrets
│   │   │   └── home-manager
│   │   │       └── secrets.yaml
│   ├── kvm-nixos-server
│   │   ├── secrets
│   │   │   ├── home-manager
│   │   │   └── nixos
├── snapshots
├── tags
└── wallpaper
```

Inside .sops.yaml I will define:
- the keys to use(and will use the public age key I copied before).
- the files with secrets and their paths relative to the root of the repository.
At this point the files with the secret(secrets.yaml do not exist).

```yaml
keys:
  - &generic_linux_distro age1t22ng09gsagym5fh5helnrs7t2lgfqdmx84ww009ndwwnzz8j3vq4k723r
  - &kvm-nixos-server age1t22ng09gsagym5fh5helnrs7t2lgfqdmx84ww009ndwwnzz8j3vq4k723r
creation_rules:
  - path_regex: machines/generic_linux_distro/secrets/home-manager/secrets.yaml$
    key_groups:
      - age:
        - *generic_linux_distro
  - path_regex: machines/kvm-nixos-server/secrets/nixos/secrets.yaml$
    key_groups:
      - age:
        - *kvm-nixos-server
  - path_regex: machines/kvm-nixos-server/secrets/home-manager/secrets.yaml$
    key_groups:
      - age:
        - *kvm-nixos-server
```

### Creating and working with secret files
This is the step where we will be creating the secrets files, that will be used
later by sops-nix.


### Integrating the secret files into nix configurations using sops-nix
Note at this point all we have are structured secret files that are encrypted with
the age private key we have previously created. From the context of our repository and
specially nix configuration those secrets does not exist, and all you can do with them
is edit them using the sops command. The next step is to use sops-nix to make them part
of the nix configuration.


### integrating sops into your repository
### integrating sops-nix into your flake
Go over the difference between home-manger and system wide and how
you will need to duplicate the keys.txt or not.

# use cases
use my repository to highlight specific examples of use case

# Bonus using with nixosanywhere

