# Acer Aspire A315-34 — NixOS beside Omarchy, with Limine retained

This is an adaptation of the uploaded dotfiles repository, not a generic replacement desktop. It keeps the existing `nectarine-tree` flake host name and most of the original aspects, but changes storage, hardware discovery, and boot installation for this Acer.

## Safety model

The layout is designed around the disk facts supplied before this guide was generated:

- `/dev/nvme0n1p1`: existing 2 GiB FAT32 EFI partition, UUID `A282-91EC`, mounted by Omarchy at `/boot`.
- `/dev/nvme0n1p2`: existing Omarchy LUKS2 partition. **Do not format, resize, relabel, or run Disko against it.**
- approximately 132 GiB unallocated after partition 2.

NixOS gets a new partition in that unallocated area only (expected `/dev/nvme0n1p3`). Its contents will be LUKS2 → LVM VG `pool` → 16 GiB swap LV + Btrfs root LV → `root`, `persist`, and `nix` subvolumes. Those names are intentionally compatible with the original impermanence module.

The existing ESP is shared, but the NixOS hook writes only to `/boot/EFI/NixOS/`. It verifies that `/boot` is the ESP with UUID `A282-91EC`. It does not install Limine, systemd-boot, GRUB, or change EFI NVRAM variables. Limine remains the top-level boot menu.

**Do not run the README's original `nectarine-gardener -- plant` command. Do not include `den.aspects.disko` for this host.** The original `nectarine-tree` Disko definition repartitions the whole disk, formats its own EFI partition, and allocates the remainder to its LUKS/LVM design.

## What was adapted

- Replaced the host's whole-disk Disko aspect with `acer-storage.nix`.
- Removed Secure Boot/Lanzaboote from the host and user aspects.
- Added NixOS's `boot.loader.external` hook to publish the kernel and initrd under `/EFI/NixOS/`, without installing another bootloader.
- Removed the Lenovo-specific `hardware.facter.reportPath` and the fixed 1920×1080 Lenovo/external-monitor XML.
- Enabled redistributable firmware and Intel microcode for the Acer.
- Kept the original flake and `flake.lock` unchanged, and retained the original desktop/user aspects, impermanence layout, services, packages, and encrypted secrets.

The new Limine entry uses Limine's Linux protocol (`path` for the kernel and `module_path` for the initrd). Limine 12 documents this protocol and supports loading resources from the ESP. See <https://github.com/limine-bootloader/limine/blob/v12.x/CONFIG.md>.

## Before installing

Boot a current graphical NixOS installer USB in UEFI mode. In the live environment, re-check the disk. These are read-only:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,PARTLABEL,PARTUUID,UUID,MOUNTPOINTS /dev/nvme0n1
parted /dev/nvme0n1 unit GiB print free
blkid /dev/nvme0n1p1 /dev/nvme0n1p2
```

Proceed only if partition 1 is still the 2 GiB FAT32 ESP with UUID `A282-91EC`, partition 2 is still the Omarchy LUKS2 partition, and the remaining space is still unallocated. If those facts differ, stop and recalculate; do not paste the partition command below blindly.

Before changing the partition table, extract the supplied bundle to a USB drive. Set `BUNDLE` to the directory containing `dotfiles-acer-limine/` (adjust the example mount path), then perform a read-only NixOS configuration evaluation:

```bash
BUNDLE=/run/media/liveuser/USB
sudo nix --extra-experimental-features 'nix-command flakes' eval --raw \
  "$BUNDLE/dotfiles-acer-limine/dotfiles/nix#nixosConfigurations.nectarine-tree.config.system.build.toplevel.drvPath"
```

This evaluates the NixOS configuration without building or writing to the disk. **If it fails, stop before partitioning and share the complete error output.** This adaptation has not been run through `nix flake check` in the preparation environment.

## Create the new partition

The earlier partition report showed partition 2 ending at 345 GiB and free space from 345 GiB to the end of the 476.9 GiB disk. **Only if the fresh `parted ... print free` output still shows that exact boundary**, create a single new partition in that free region:

```bash
sudo parted /dev/nvme0n1 --script --align optimal mkpart NIXOS 345GiB 100%
sudo partprobe /dev/nvme0n1
lsblk -o NAME,SIZE,TYPE,FSTYPE,PARTLABEL,PARTUUID,UUID /dev/nvme0n1
```

Confirm that `/dev/nvme0n1p3` is the new ~132 GiB partition and that p1 and p2 are unchanged. If the device name or size is different, stop and do not run the next section until the device mapping is corrected.

## Encrypt and create the LVM/Btrfs layout

**The following commands erase only the new partition `/dev/nvme0n1p3`.** Read each command and verify the target before running it. Never substitute p1 or p2.

```bash
sudo cryptsetup luksFormat --type luks2 /dev/nvme0n1p3
sudo cryptsetup open /dev/nvme0n1p3 crypted

sudo pvcreate /dev/mapper/crypted
sudo vgcreate pool /dev/mapper/crypted
sudo lvcreate --yes --size 16G --name swap pool
sudo lvcreate --yes --extents 100%FREE --name root pool

sudo mkfs.btrfs -L nixos-root /dev/pool/root
sudo mkswap -L nixos-swap /dev/pool/swap
```

The 16 GiB swap LV is intended to support the original `suspend-then-hibernate` policy on an 8 GiB machine. Actual hibernation success still depends on the amount of RAM in use at suspend time.

Create the subvolumes and mount them:

```bash
sudo mkdir -p /mnt
sudo mount /dev/pool/root /mnt
sudo btrfs subvolume create /mnt/root
sudo btrfs subvolume create /mnt/persist
sudo btrfs subvolume create /mnt/nix
sudo umount /mnt

sudo mount -o subvol=root,compress=zstd,noatime /dev/pool/root /mnt
sudo mkdir -p /mnt/persist /mnt/nix /mnt/boot
sudo mount -o subvol=persist,compress=zstd,noatime /dev/pool/root /mnt/persist
sudo mount -o subvol=nix,compress=zstd,noatime /dev/pool/root /mnt/nix
sudo mount /dev/nvme0n1p1 /mnt/boot
sudo swapon /dev/pool/swap
```

Check mounts before proceeding:

```bash
findmnt /mnt /mnt/persist /mnt/nix /mnt/boot
lsblk -f /dev/nvme0n1
```

`/mnt/boot` must resolve to the existing FAT32 `/dev/nvme0n1p1`. Do not format it.

## Put the adapted dotfiles and secrets in the persistent home

Copy the `dotfiles/` folder from this bundle to `/mnt/persist/home/ruhtra/dotfiles`. For example, after extracting this bundle to a USB drive mounted at `/run/media/liveuser/USB`:

```bash
sudo mkdir -p /mnt/persist/home/ruhtra/dotfiles
sudo cp -a /run/media/liveuser/USB/dotfiles-acer-limine/dotfiles/. /mnt/persist/home/ruhtra/dotfiles/
```

Use the actual USB mount path. In the supplied ZIP, the repository is inside `dotfiles-acer-limine/dotfiles/`. Preserve the `.git` directory and hidden files; the Acer-specific Nix files are staged in that Git working tree so the local flake source includes them.

The original configuration expects the Age identity at:

```text
/persist/home/ruhtra/.local/share/pa/identities
```

Restore that private identity file from your backup before installation if you need the existing agenix secrets/services. The archive contains encrypted `.age` secret files and the public recipient configuration, but it does **not** contain the private Age identity needed to decrypt them. Do not paste or share that identity.

## Set the new LUKS UUID in NixOS config

Get the UUID from the newly formatted p3 LUKS header:

```bash
LUKS_UUID=$(sudo cryptsetup luksUUID /dev/nvme0n1p3)
echo "$LUKS_UUID"
```

Edit this file:

```text
/mnt/persist/home/ruhtra/dotfiles/nix/modules/features/acer-storage.nix
```

Replace `REPLACE_WITH_LUKS_UUID` with the UUID printed above, without angle brackets or quotes beyond the existing string quotes. The `device` line must become:

```nix
device = "/dev/disk/by-uuid/PASTE-THE-UUID-HERE";
```

Do not use the PARTUUID here; it must be the LUKS UUID from `cryptsetup luksUUID`.

## Install NixOS

The flake is in the repository's `nix/` directory. Run:

```bash
sudo env NIX_CONFIG="experimental-features = nix-command flakes" nixos-install --flake /mnt/persist/home/ruhtra/dotfiles/nix#nectarine-tree --no-root-passwd
```

This is a large configuration (Niri/Sway, GNOME, COSMIC, virtualization, many custom packages) and the Acer has 8 GiB RAM. Expect the build to be slow; the 16 GiB swap LV is enabled above to give the installer more headroom.

The external boot hook should create:

```text
/boot/EFI/NixOS/vmlinuz
/boot/EFI/NixOS/initrd
```

Before rebooting, check that both files exist and are non-empty:

```bash
sudo ls -lh /mnt/boot/EFI/NixOS/
```

If they are absent, do not reboot into NixOS; inspect the `nixos-install` error first.

## Add NixOS to Limine

Back up the current Limine config, then append a new top-level entry. The file is `/mnt/boot/limine.conf` from the installer, or `/boot/limine.conf` after booting Omarchy:

```bash
sudo cp -a /mnt/boot/limine.conf /mnt/boot/limine.conf.before-nixos
sudo tee -a /mnt/boot/limine.conf >/dev/null <<EOF

/NixOS (Acer)
    comment: NixOS on Acer Aspire A315-34
    protocol: linux
    path: boot():/EFI/NixOS/vmlinuz
    module_path: boot():/EFI/NixOS/initrd
    cmdline: init=/nix/var/nix/profiles/system/init rd.luks.name=${LUKS_UUID}=crypted rd.lvm.vg=pool root=/dev/mapper/pool-root rootfstype=btrfs rootflags=subvol=root rw resume=/dev/pool/swap quiet splash plymouth.use-simpledrm loglevel=3 rd.systemd.show_status=false rd.udev.log_level=3 udev.log_priority=3
EOF
```

In this shell, set `LUKS_UUID` again to the actual UUID if it is not still defined in the current shell:

```bash
LUKS_UUID=$(sudo cryptsetup luksUUID /dev/nvme0n1p3)
```

Append the NixOS entry after the existing Omarchy and EFI fallback entries. Do not change `default_entry`, `timeout`, the Omarchy entry, the fallback entry, or any EFI loader file. Verify the result:

```bash
sudo tail -n 18 /mnt/boot/limine.conf
sudo ls -lh /mnt/boot/EFI/BOOT/BOOTX64.EFI /mnt/boot/EFI/limine/limine_x64.efi /mnt/boot/EFI/Linux/omarchy_linux-omarchy.efi /mnt/boot/EFI/NixOS/
```

The two Omarchy EFI files and Limine loader should still exist unchanged. You should now see the NixOS entry in Limine. Leave Omarchy's existing default entry unchanged.

On later NixOS bootloader updates the hook retains the previous kernel/initrd pair as `vmlinuz.previous` and `initrd.previous`. These are recovery copies; they are not automatically added to Limine.

## First boot and known caveats

1. Reboot and choose `NixOS (Acer)` in Limine. Keep Omarchy as the default until NixOS has booted successfully several times.
2. Check Wi-Fi (Intel `iwlwifi`/IWD), Ethernet (Realtek `r8169`), graphics (`i915`), audio, suspend, and resume.
3. Confirm that the Age identity was restored and agenix secrets activate. The config's home persistence depends on `/persist`; do not remove that LV or its subvolume.
4. The old `nectarine-tree.json` hardware report and fixed monitor XML belong to another laptop and are no longer imported.
5. The upstream `nectarine-tree` host still keeps its original hostname. The NixOS install command uses the flake output name `nectarine-tree`.

If NixOS fails to boot, use Limine to select Omarchy. Do not attempt automatic repair commands that touch the partition table or the ESP; collect the error message and current `lsblk -f`, `cryptsetup status crypted`, and `findmnt` output first.
