+++
title = 'Keeping Nix Secrets with Sops: Integration and Applications'
date = 2025-06-29T18:47:34+03:00
draft = false
tags = ["sops", "sops-nix", "age", "nixos", "nix", "home-manager"]

[cover]
image = "cover.jpg"
alt = "NixOS logo with lock"
relative = true
+++
When setting up a full NixOS system—or even just a standalone Home Manager module—secrets are often a core part of the
configuration. With nix being declarative, it makes a lot of sense to integrate them into the nix configurations, but
this needs to be done in a secure manner, so even if your configurations are put on public display your secrets will
stay safe(case in point [my repository](https://github.com/p3t33/nixos_flake) that are a reference for this post).

This integration is also the final step for a single line provisioning of a new system (using a tool such as
[nixos-anywhere](https://github.com/nix-community/nixos-anywhere)), that once booted will require no additional work to be made fully operational.

This post will cover the integration of [sops-nix](https://github.com/Mic92/sops-nix) into NixOS/Home-Manger and its various
applications based on my use cases. With the main goal of keeping a healthy balance between getting things up and
running from start to finish(by going over all the moving parts and gluing all of them together), while avoiding
repeating sops-nix readme and repository implementation.

Even if you already have sops-nix running I hope my application use cases will give you new ideas for use in your
own configurations.

<u>As I am using a flake this post will reflect that</u>

## Anatomy Of Sops With nix
There are a few moving parts that come together to allow the use of sops with nix via sops-nix module.

- age: An encryption tool that provides a private–public key pair for encrypting
and decrypting files securely and simply.
- sops:  A secrets management tool that uses age (among other options like GPG, KMS, etc.)
to encrypt and decrypt structured configuration files, such as YAML, JSON, and .env files.
- sops-nix: A NixOS/Home Manager module that integrates sops into system configuration.
It securely decrypts and makes SOPS-managed secrets available at runtime for use in services, environment
variables, or other system paths.

From this description it is clear that including sops-nix into your flake input by itself is pointless
and for it to work, you will need to do more things, such as create age key pair and, use sops and nix to
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

And in the case of stand alone home-manager module

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
activation, at boot time.

### Creating sops configuration file
sops as cli tool operates on secret files, it looks for private keys in expected paths,
such as ~/.config/sops/age/keys.txt. It can be used with bunch of arguments or it can be used with a
configuration file that will make life easier.

There are 3 main things to understand here:
1. sops configuration file is not the actual secret, it is just a convenience to avoid typing(paths, keys ...)and is not required by sops-nix.
2. sops will look for the public key you state in your sops configuration file in ~/.config/sops/age/keys.txt.
3. sops automatically searches upward in the directory tree for a file named .sops.yaml, which is the name of the
configuration file.

At the root of my nix configuration repository I will create .sops.yaml so the partial
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
This is the step where we will be creating the files with secrets, that will be used
later by sops-nix. I will be using yaml.

At the path that was defined inside .sops.yaml(E.g machines/kvm-nixos-server/secrets/nixos)
Execute(use same file name as in .sops.yaml):

```bash
sops secrets.yaml
```
You will get a file with some boiler plate inside of it, for you to edit.
The syntax is very simple, and follows yaml. And can be roughly divided into
3 types:

```yml
username_password: "mypassword"

some_service:
    username: "some_username"
    password: "mypassword"

ssh_key: |
    <private key content>
```

Once you are done, you can save and exist, if every thing worked you should see
encrypted data if you try to open the secrets.yaml file without using sops command.
And once you execute "sops secrets.yaml" you should be able to see the decrypted
data and edit the file.

Once you use each individual secret(inside of your nix configuration) it will be created
as a file under /run/secrets.d/


### Integrating sops-nix into nix flake
Note that at this point all we have are some structured files with secrets that are encrypted
with the age private key we have previously created. Files we can encrypt and decrypt
secrets from the CLI. From the context of our repository and specially nix configuration those
secrets are not yet integrated into the nix configuration. The next step is to setup sops-nix, and with it we well be able
to use the secrets we created.

The following part will show snippets form my repository and for full code you should look
at [my repository](https://github.com/p3t33/nixos_flake) to get the full picture.

Integration start with adding sops-nix as a flake input(flake.nix)
```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-25.05";

    home-manager = {
      url = "github:nix-community/home-manager/release-25.05";
      inputs.nixpkgs.follows = "nixpkgs";
    };

    sops-nix = {
      url = "github:Mic92/sops-nix";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };
}
```

#### NixOS with home-manager as a module
Inside flake.nix
For the NixOS level, we add into the flake
```nix
nixosConfigurations = {
  kvm-nixos-server = lib.nixosSystem {
    inherit system;
    specialArgs = {
      inherit inputs;
      machineName = "kvm-nixos-server";
    };
    modules = [
      ./machines/kvm-nixos-server/configuration.nix
      inputs.home-manager.nixosModules.home-manager
      inputs.sops-nix.nixosModules.sops
    ];
  };
};
```

Next addition is done inside of the home-manager configuration files
```nix
{
  inputs,
  ...
}:
{
  home-manager = {
    sharedModules = [
      inputs.sops-nix.homeManagerModules.sops
    ];
  };
}
```

#### home-manager as a stand alone module(used with generic Linux such as Fedora)
```nix
homeConfigurations = {
  kmedrish = inputs.home-manager.lib.homeManagerConfiguration {
    inherit pkgs;
    extraSpecialArgs = {
      inherit inputs;
      machineName = "generic_linux_distro";
    };
    modules = [
      ./machines/generic_linux_distro/home.nix
      inputs.sops-nix.homeManagerModules.sops
    ];
  };
};
```
### setting defaults for sops-nix
This part is not a must, but it makes sense if you would like to avoid repeating
parts of the declarations for every secret you decrypt using sops-nix.

This configuration need to be set for NixOS and for home-manager meaning that
if, like me, you are using home-manager as NixOS module you will be defining
them twice in your configuration.

Those default will define a secrets file and its path, the format
of the file, and the secret age key and its path that will be used to decrypt
secrets. After that all you will need to do is to define the secret inside
of the secrets file and sops-nix will be able to do the rest.

The configuration are:

For NixOS:
```nix
{ config, ... }:
{
  sops.defaultSopsFile = "../machines/kvm-nixos-server/secrets/nixos/secrets.yaml";
  sops.defaultSopsFormat = "yaml";
  sops.age.keyFile = "/path_to_/keys.txt";
}
```

For home-manager
```nix
{ config, ... }:
{
  sops.defaultSopsFile = "../machines/kvm-nixos-server/secrets/home-manager
  sops.defaultSopsFormat = "yaml";
  sops.age.keyFile = "/path_to_/keys.txt";
}
```
<u>Note:</u> the defaults share the same secret age key but, each having a separate
secrets.yaml file.


## Use cases
At this point sops-nix should be integrated into your nix flake, and now all that is
left is to start using the secrets encrypted with sops and age by declaring them
in our nix configuration. This section is intended to give you ideas on how you
could utilize sops.

This part will also show some basic syntax of how to use sops-nix module but your
best bet is to look at [my repository](https://github.com/p3t33/nixos_flake) for a working reference.

### keeping secret keys
The most basic use case I can think of are secret keys, be those private ssh keys you
want to be ready after you install the system or secret keys used by syncthing.

You will start by defining the keys inside the secrets.yaml

```yaml
syncthing:
    cert.pem: |
        -----BEGIN EC PRIVATE KEY-----
        <some hash>
        -----END CERTIFICATE-----


    key.pem: |
        -----BEGIN EC PRIVATE KEY-----
        <some hash>
        -----END CERTIFICATE-----
```

And then inside of your syncthing.nix file you will start by making the secrets available:

```nix
sops.secrets."syncthing/cert.pem" = {
  owner = "<user name syncthing is running as>";
  mode = "0600";

};

sops.secrets."syncthing/key.pem" = {
  owner = "<user name syncthing is running as>";
  mode = "0600";
};
```

And finish by setting their path for the syncthing service

```nix
services.syncthing = {
  enable = true;
  key = config.sops.secrets."syncthing/key.pem".path;
  cert = config.sops.secrets."syncthing/cert.pem".path;
};
```

### Including as a file with Secrets
Not all of the secrets can be put into a "box" in the form of nix configuration
such as services.syncthing.key = ... Some times you will be using include directives
that are native to the module you are configuring.

The following two examples aren't strictly secrets but are information I rather not to
share via my public repository.

#### ssh client
As much as I like to define hosts to use with ssh client I would prefer not to share the information
I use to connect to them on my public repository. The solution is to use the Include directive inside of
~/.ssh/config

I start with defining my hosts in my secrets.yaml in the same syntax which the native ~/ssh/config is expecting them:

``` yaml
extra_hosts: |
    Host 1-dev
        HostName <ip>
        User <user name>
        IdentitiesOnly yes
        IdentityFile <private ssh key path>

    Host 2-dev
        HostName <ip>
        User <user name>
        IdentitiesOnly yes
        IdentityFile <private ssh key path>
```

I then define the path to the this file inside of the ssh client configuration

```nix
    programs.ssh = {
      enable = true;
      extraConfig = "
        # This file will be generated with sops and if sops fails to generate
        # it this directive will be skipped.
        Include ${config.sops.secrets.extra_hosts.path}
        ";
    };
```

This allows ~/.ssh/config to be defined in my configuration and extended by
its own Include directive with my decrypted secrets.

#### git
In a similar matter to ssh, we start by defining a secret that will have git information
that should be kept out of the public configuration repository.


```yaml
git: |
    [user]
      email = "<...>"
      name = "<...>"
      signingkey = <...>
```

And then in the git.nix configuration, include can be use to access the information
inside of the secrets file that will be generated.

```nix
programs.git = {
  enable = true;
  extraConfig = {
    include = {
      path = config.sops.secrets.git.path;
    };
  };
};
```

### Linux user password
User password can set for any user and kept as a secret using sops.

You start be creating the password in the terminal

```bash
read -s password && echo "$password" | mkpasswd -s
```

Then you define this password as a secret
```yaml
initial_hashed_password: <...>
```

Next it is defined for the desired user

```nix
sops.secrets.initial_hashed_password.neededForUsers = true;

users.users.<...> = {
  isNormalUser = true;
  hashedPasswordFile = config.sops.secrets.initial_hashed_password.path;
  description = hostSpecific.primeUsername;
  extraGroups = [
    "networkmanager"
    "wheel"
  ];
};
```

### NetworkManger WiFi profiles using environment variables
Environment file is just a file with multiple lines starting with variable name and assigning a value to it.

```text
FOO=bar
MEAT=BALL
```

This file is passed by systemd to the service and the variables inside it act as part of the service environment
variables in the same way as $HOME is environment variable inside of the shell.

It is very convenient to set your WiFi SSID and password as part of your NixOS configuring
and this can be done using environment variables that are set inside of sops secret file that
is provided to NetworkManger as environmentFile to be used as part of its profiles.

You start by creating a sops secret

```yaml
wifi:
    home:
        ssid: HOME_SSID=<...>
        psk: HOME_PSK=<...>
```

Note that this time the secret is actually a variable name and a secret that is assigned to it.
From the context of sops this is just a string. But once this secret is decrypted two files will
be created /run/secrets/wifi/psk and /run/secrets/wifi/SSID

Each file will have a single line

```text
HOME_SSID=<...>
```

And

```text
HOME_PSK=<...>
```

Next we can use those variables inside of the NetworkManger configuring file

```nix
{ config, ... }:
{
  sops.secrets."wifi/home/ssid" = {};

  sops.secrets."wifi/home/psk" = {};

  networking.networkmanager = {

    ensureProfiles = {

      environmentFiles = [
        config.sops.secrets."wifi/home/ssid".path
        config.sops.secrets."wifi/home/psk".path
      ];

      profiles = {
        "home" = {
          connection = {
            id = "home";
            type = "wifi";
            autoconnect = true;
          };
          wifi = {
            mode = "infrastructure";
            ssid = "$HOME_SSID";
          };
          wifi-security = {
            key-mgmt = "wpa-psk";
            psk = "$HOME_PSK";
          };
          ipv4 = {
            method = "auto";
          };
          ipv6 = {
            method = "auto";
            addr-gen-mode = "stable-privacy";
          };
        };
      };
    };
  };
}
```
Note that special syntax is used in the NetworkManger to access the variables that
were set using the environmentFile(E.g $HOME_SSID). This syntax might change based on
the services you are configuring(E.g when using the same environmentFile mechanism with
gatus and homepage dashboard).

# Bonus using with nixos-anywhere
I am a big fan of [nixos-anywhere](https://github.com/nix-community/nixos-anywhere), as it allows me to install my entire NixOS
system with a single command. sops-nix complicates this a bit as the secret key we
generated at the very beginning of this post needs to be preset during installation and
during the first boot. The way I am using it now is to put the key,
into /keys.txt(owned by root and with strict permissions to access it).


You start by setting a default path for the key

```nix
{ config, inputs, hostSpecific, ... }:
{
  sops = {
    defaultSopsFile = inputs.self + "/machines/${hostSpecific.hostName}/secrets/nixos/secrets.yaml";
    defaultSopsFormat = "yaml";
    age.keyFile = "/keys.txt";
  };
}

```
As already mentioned, this needs to be done twice(for home-manager as nixos module as well).

```nix
{ config, inputs, hostSpecific, ... }:
{
  sops.defaultSopsFile = inputs.self + "/machines/${hostSpecific.hostName}/secrets/home-manager/secrets.yaml";
  sops.defaultSopsFormat = "yaml";
  sops.age.keyFile = "/keys.txt";
}
```

Next when executing the nixos-anywhere command you will specify(as an extra file) the directory where
the keys.txt is located, E.g:

```bash
nix run github:nix-community/nixos-anywhere/1.9.0  -- --flake .#<...> --extra-files /your_path/sops-kvm root@<ip of target>
```

