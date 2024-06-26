```txt
/nix            /                       /persist
            +------------------------+
  /store/x -|-> /var/x               |
            |   /var/lib/bluetooth <-|-   /var/lib/bluetooth
  /store/y -|-> /bin/y               |
            |   /home/admin/.local <-|-   /home/admin/.local
  /store/z -|-> /etc/z               |
            +------------------------+
```

```nix
{...}: {
  boot.initrd.postDeviceCommands = lib.mkAfter ''
    mkdir /btrfs_tmp
    mount /dev/mapper/crypted /btrfs_tmp
    if [[ -e /btrfs_tmp/root ]]; then
        mkdir -p /btrfs_tmp/old_roots
        timestamp=$(date --date="@$(stat -c %Y /btrfs_tmp/root)" "+%Y-%m-%-d_%H:%M:%S")
        mv /btrfs_tmp/root "/btrfs_tmp/old_roots/$timestamp"
    fi

    delete_subvolume_recursively() {
        IFS=$'\n'
        for i in $(btrfs subvolume list -o "$1" | cut -f 9- -d ' '); do
            delete_subvolume_recursively "/btrfs_tmp/$i"
        done
        btrfs subvolume delete "$1"
    }

    for i in $(find /btrfs_tmp/old_roots/ -maxdepth 1 -mtime +30); do
        delete_subvolume_recursively "$i"
    done

    btrfs subvolume create /btrfs_tmp/root
    umount /btrfs_tmp
  '';
}
```

```nix
{
  inputs = {
    impermanence = {
      url = "github:nix-community/impermanence";
    };
  };

  outputs = {self, ...} @ inputs: let
    system = "x86_64-linux";
    pkgs = inputs.nixpkgs.legacyPackages.${system};
  in {
    nixosConfigurations = {
      default = inputs.nixpkgs.lib.nixosSystem {
        inherit system;

        modules = [
          inputs.impermanence.nixosModules.impermanence
          ...
        ];
      };
    };
  }
}
```

```nix
{...}: {
  fileSystems."/persist".neededForBoot = true;
  environment.persistence."/persist" = {
    hideMounts = true;
    directories = [
      {
        directory = "/etc/nixos";
        user = "demo";
        mode = "u=rwx,g=rx,o=rx";
      }
      "/var/log"
      "/var/lib/bluetooth"
      "/var/lib/nixos"
      "/var/lib/systemd/coredump"
      "/etc/NetworkManager/system-connections"
    ];
    files = [
      "/etc/machine-id"
      {
        file = "/var/keys/secret_file";
        parentDirectory = {mode = "u=rwx,g=,o=";};
      }
    ];

    users."demo" = {
      directories = [
        "this-will-persist"
      ];
    };
  };
}
```
