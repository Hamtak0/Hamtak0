# NIXOS Installation guide

## Model structure

An example model of a disk structure that was used in this installation guide
```
nvmeXn1                     512GB
├─nvmeXn1p1 -> EFI          1GB
├─nvmeXn1p2 -> swap         16GB (Ram 16GB)
└─nvmeXn1p3 -> lvm          REMAIN
               ├─ nix_root  128GB
               └─ nix_home  300GB
```

## Install Nix

Connect to internet via nmcli  
```
$ ip a
$ nmcli device wifi list
$ nmcli device wifi connect "SSID_or_BSSID" password "PASSWORD"
$ nmcli connection show
```

(Extra) Connect to KUWIN internet
```
$ nmcli con add \
    type wifi \
    ifname <IFNAME> \
    con-name KUWIN \
    ssid KUWIN \
    ipv4.method auto \
    802-1x.eap peap \
    802-1x.phase2-auth mschapv2 \
    802-1x.identity <IDENTITY> \
    802-1x.password <PASSWORD> \
    wifi-sec.key-mgmt wpa-eap
```
> IFNAME = wlan (example)
>
> use `nmcli connection show` to choose wifi IFNAME

Login into root
```
$ sudo -i
```

Partitioning the disk
```
$ lsblk
$ fdisk -l /dev/nvmeXn1
$ cfdisk /dev/nvmeXn1
```

Formatting boot and swap
```
$ mkfs.fat -F 32 -n boot /dev/nvmeXn1p1
$ mkswap -L swap /dev/nvmeXn1p2
```

Create logical volume manager (lvm)
```
// Physical volumes
$ pvcreate
$ pvs

// Volume groups
$ vgcreate nix_vg /dev/nvmeXn1p3
$ vgs

// Logical volumes
$ lvcreate -L 128GB nix_vg -n nix_root
$ lvcreate -L 300GB nix_vg -n nix_home

// Load kernel module
$ modprobe dm_mod

// Activate
$ vgscan
$ vgchange -a y nix_vg
```

Formatting root and home
```
$ mkfs.ext4 /dev/nix_vg/nix_root
$ mkfs.ext4 /dev/nix_vg/nix_home
```

Mount the target file system
```
// mount root
$ mount /dev/nix_vg/nix_root /mnt

// mount home
$ mkdir -p /mnt/home
$ mount /dev/nix_vg/nix_home /mnt/home

// mount boot
$ mkdir -p /mnt/boot
$ mount -o umask=077 /dev/disk/by-label/boot /mnt/boot

// mount swap
$ swapon /dev/nvmeXn1p2
```

Generate nixos config file
```
$ nixos-generate-config --root /mnt
// BAN THE NANOS
$ vim /mnt/etc/nixos/configuration.nix
```

Examples of nix configuration
```
{ config, pkgs, ... }: {
    imports = [
        # Include the results of the hardware scan.
        ./hardware-configuration.nix
    ];

    boot.loader.systemd-boot.enable = true;

    # Note: setting fileSystems is generally not
    # necessary, since nixos-generate-config figures them out
    # automatically in hardware-configuration.nix.
    # fileSystems."/".device = "/dev/disk/by-label/nixos";

    # Enable Network Manager
    networking.networkmanager.enable = true;

    # Users
    users.users = {
        tako = {
          isNormalUser = true;
          description = "Hamtako";
          extraGroups = ["wheel" "networkmanager"];
        };
      };

    # Enable the OpenSSH server.
    # services.sshd.enable = true;
}
```

Do the installation
```
$ nixos-install
```

If everything went well
```
$ reboot
```

### NOW YOU HAVE YOUR NIXOS (THE FUTURE OS)

## References

[1] [Nixos manual](https://nixos.org/manual/nixos/stable/#ch-installation)  
[2] [Arch dual boot](https://gist.github.com/vortexavalanche/64b3a7b97b3f163e252c49d6f82e6151)  
[3] [Network Manager](http://wiki.archlinux.org/title/NetworkManager)  
[4] [LVM](https://wiki.archlinux.org/title/LVM)