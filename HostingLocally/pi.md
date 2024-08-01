# Hosting on the Web without the Fluff


## Project Overview

I have a small rust web app which can be found [here](https://github.com/NitroSniper/ortin.git) which is using axum on the backend and just serve static pages rn.
I want to host this service on the web, However I don't have a continuous income that I can burn on a EC2 instance.

So I'm gonna host this on a second hand rasberry pi 3B+ I found at Cex, which doesn't have the best performance.
If I want my web app to be peformant, I will need to reduce as much fluff while still mataining core logic.

I'm already writing in Rust which is already a performant language but I need to wrap this project as a Docker Image since it is really easy to distribute.
However if I to create this via a crude Dockerfile 


```dockerfile
FROM rust:1.67 as builder
WORKDIR /usr/src/myapp
COPY . .
RUN cargo install --path .

FROM debian:bullseye-slim
RUN apt-get update && rm -rf /var/lib/apt/lists/*
COPY --from=builder /usr/local/cargo/bin/myapp /usr/local/bin/myapp
CMD ["myapp"]
```


!!!!!!!!!!!!!!!!!!!TODO!!!!!!!!!!!!!

This image is quite large and contains some fluff that we don't need. However I know a way to reduce it with a tool that is powering my entire workstation, **Nix**

## Creating a Nix Flake

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";

    # rust dev toolchain
    rust-overlay = {
      url = "github:oxalica/rust-overlay";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs =
    { ... }@inputs:
    inputs.flake-utils.lib.eachDefaultSystem (
      system:
      let
        overlays = [ (import inputs.rust-overlay) ];
        pkgs = import inputs.nixpkgs { inherit system overlays; };

        # Binary name of cargo project
        name = "ortin";

        # Build rust binary 
        bin = (
          pkgs.rustPlatform.buildRustPackage {
            inherit name;
            cargoLock.lockFile = ./Cargo.lock;
            src = pkgs.lib.cleanSource ./.;
            postInstall = ''
              mkdir app/
              mv resources/ $out/
            '';
          }
        );

        # Build Docker Image
        docker = pkgs.dockerTools.buildImage {
          inherit name;
          tag = "latest";

          copyToRoot = [ bin ];

          config = {
            Cmd = [ "/bin/${name}" ];
            WorkingDir = "/";
          };
        };
      in
      with pkgs;
      {
        packages = {
          inherit bin docker;
          default = bin;
        };


        # Custom dev shell with nix develop
        devShells.default = mkShell {
          buildInputs = [
            (rust-bin.stable.latest.default.override {
              extensions = [
                "rust-src"
                "rustfmt"
                "rust-analyzer"
              ];
            })
            bacon
            tailwindcss
            hey
            watchexec
            djlint
            # dive 
          ];
        };
        env = {
          # Required by rust-analyzer
          RUST_SRC_PATH = "${pkgs.rustToolchain}/lib/rustlib/src/rust/library";
        };
      }
    );
}
```

This Nix flake seems massive but is very versatile
It is 
* A developer environment with `nix develop` which installs a bunch of packages that I like when developing and the rust toolchain
* A way to compile the project into a single binary which relies on the developer system via `nix build` _(default)_
* A way to package that binary into a Docker Image via `nix buid .#docker` which targets the docker output

and that last part isn't just a basic docker image. It builds the **minimal** docker image that is needed for the binary to run. 

Nothing more, Nothing less. 

**That resulted in the image size being 45.8 MB**


Now time to run this image on the RasberryPi, But we need to install a OS onto it and configure it to run the website.
We could do this with a OS like debian or Ubuntu but it's imperative configuration and requiring a bit of manual setup as well as not being able to lock certain packages causes some issue if I want to access this project 3 years later. 
But NixOS is none of that, and I already am using Nix so it shouldn't be too hard

## Installing NixOS on the Pi

First we want to grab a [NixOS image for ARM](https://hydra.nixos.org/job/nixos/trunk-combined/nixos.sd_image.aarch64-linux)
which I got `https://hydra.nixos.org/build/265833086` which gives me this [download link](https://hydra.nixos.org/build/265833086/download/1/nixos-sd-image-23.11.7870.205fd4226592-aarch64-linux.img.zst)

> **Note:**  When picking the latest version, it can have some issues due to the nature of it being bleeding edge. 
> 
> I suggest looking at this [page](https://nixos.wiki/wiki/NixOS_on_ARM#Installation) and pick one of the stable releases if you want to minimise possible issues happening. Or you can just download the same one as mine. NixOS is meant for having software to last.

We want to download the file and then decompress the `.zst` file. 
There are many ways to do this but I'm using `nix-shell` since I'm already used to it

![](https://raw.githubusercontent.com/NitroSniper/Blogs/main/HostingLocally/getting_hyda_img.png "Downloading the NixOS image")


Now we have the `.img` file and now we just need to burn/flash it onto a SD card. You can use various differnt software such as [Etcher](https://etcher.balena.io).
However, for my preference, I'm using a tool called [caligula](https://github.com/ifd3f/caligula.git)

![](https://raw.githubusercontent.com/NitroSniper/Blogs/main/HostingLocally/burning_SD_card.gif "Burning the NixOS image onto the SD card")


**_Now we have a SD card with NixOS installed._**

## Setting up the Pi with NixOS

### Configuring via NixOS configuration.nix file
I suggest connecting the RBpi with an Ethernet tempoarily but if you can't, you have to connect it via `wpa_cli`. I'm not gonna explain it, even tho I did it but you can find the resource I used [here](https://vmware.github.io/photon/docs-v5/administration-guide/configure-wireless-networking/)

> #### If you have a small amount of RAM
> It maybe smart to make a tempoary swap file since sometimes building the NixOS requires a bit of RAM
> 
> 1. Creation of 4GB empty file
> `sudo dd if=/dev/zero of=/swapfile bs=1M count=4096`
> 2. Give the empty file root permission
> `sudo chmod 600 /swapfile`
> 3. Make the empty file a swap file
> `sudo mkswap /swapfile`
> 4. Turn on the swap file
> `sudo swapon /swapfile`
> 
> Once we are done with it or it is not needed. we can just remove it via
> ```bash
> sudo swapoff /path/to/file
> sudo rm /path/to/file
> ```
> 
> 
> Now that we have that sorted, we want to create our config file for the Pi


```nix
{ config, lib, pkgs, ... }:

{
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
    ];

  boot.loader.grub.enable = false;
  boot.loader.generic-extlinux-compatible.enable = true;

  networking.hostName = "nixos"; # Define your hostname.
  networking.networkmanager.enable = true;  # Easiest to use and most distros use this by default.

  # Set your time zone.
  time.timeZone = "Europe/Amsterdam";


  i18n.defaultLocale = "en_GB.UTF-8";
  console = {
    font = "Lat2-Terminus16";
    keyMap = "uk";
  };

  # Configure keymap in X11
  services.xserver.xkb.layout = "gb";

  # Define a user account. Don't forget to set a password with ‘passwd’.
  users.users.ortin = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" "docker" ]; # Enable ‘sudo’ for the user.
    packages = with pkgs; [
    ];
  };

  virtualisation.docker = {
    enable = true;
    rootless = {
      enable = true;
      setSocketVariable = true;
    };
  };
  
  environment.systemPackages = with pkgs; [
    neovim # Do not forget to add an editor to edit configuration.nix! The Nano editor is also installed by default.
    wget
    git
    docker
    docker-compose
    
  ];


  # Enable the OpenSSH daemon.
  services.openssh.enable = true;

  # Open ports in the firewall.
  networking.firewall = {
    enable = true;
    allowedTCPPorts = [ 80 443 22 ];
  };

  # Copy the NixOS configuration file and link it from the resulting system
  # (/run/current-system/configuration.nix). This is useful in case you
  # accidentally delete configuration.nix.
  # system.copySystemConfiguration = true;

  # This option defines the first version of NixOS you have installed on this particular machine,
  # and is used to maintain compatibility with application data (e.g. databases) created on older NixOS versions.
  #
  # Most users should NEVER change this value after the initial install, for any reason,
  # even if you've upgraded your system to a new NixOS release.
  #
  # This value does NOT affect the Nixpkgs version your packages and OS are pulled from,
  # so changing it will NOT upgrade your system.
  #
  # This value being lower than the current NixOS release does NOT mean your system is
  # out of date, out of support, or vulnerable.
  #
  # Do NOT change this value unless you have manually inspected all the changes it would make to your configuration,
  # and migrated your data accordingly.
  #
  # For more information, see `man configuration.nix` or https://nixos.org/manual/nixos/stable/options#opt-system.stateVersion .
  system.stateVersion = "23.11"; # Did you read the comment?
}
```







## Prelude
I have a simple rust application that I want to host on the web that can be found [here](https://github.com/NitroSniper/ortin.git). 

main.rs
```rust
use askama_axum::Template;
use axum::{routing::get, Router};
use tokio::fs;
use tower_http::services::ServeDir;

#[derive(Template)]
#[template(path = "blog.html")]
struct BlogTemplate {
    content: String,
}

#[derive(Template)]
#[template(path = "home.html")]
struct HomeTemplate<'a> {
    dark: bool,
    text: &'a str,
}

#[tokio::main]
async fn main() {
    let router = Router::new()
        .route(
            "/",
            get(|| async {
                HomeTemplate {
                    dark: false,
                    text: "# Hello World",
                }
            }),
        )
        .route(
            "/blog",
            get(|| async {
                BlogTemplate {
                    content: markdown::to_html(
                        &fs::read_to_string("src/pi.md")
                            .await
                            .expect("to be a markdown file"),
                    ),
                }
            }),
        )
        .nest_service("/static", ServeDir::new("static"))
        .layer(tower_livereload::LiveReloadLayer::new());

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8000").await.unwrap();
    axum::serve(listener, router).await.unwrap();
}
```
This is great but it is only hosted locally. I want this published on the web so first we need to setup a server. 

## Setting up a Server
I chose NixOS because it robust and minimal and I will be hosting this on a RasberryPi

### Installing NixOS on RasberryPi







