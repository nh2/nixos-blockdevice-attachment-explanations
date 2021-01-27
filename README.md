# Block device file-system auto-creation on NixOS

This short article explains how it works that for a NixOps machine you can just [write](https://github.com/NixOS/nixops-aws/blob/72d0a4c812c6b562ab6a5a18cf98e66193958897/examples/trivial-ec2-ebs.nix#L9-L17):

```nix
{
  # Mount a 1 GiB EBS volume on /data.  It's created and formatted
  # when the machine is deployed, and destroyed when the machine
  # is destroyed.
  fileSystems."/data" = {
    autoFormat = true;
    fsType = "btrfs";
    device = "/dev/xvdf";
    ec2.size = 1;
  };
}
```

and an AWS EC2 EBS disk is automatically created, formatted, auto-resized, and mounted.

It also explains how you can control what happens when an EBS volume (or any other block device) is attached at runtime, to get auto-formatting or auto-mounting.

This was written for NixOS 20.09.


## How it works

* Example of NixOps automatic EBS data volume creation (`fileSystems."/data" = { autoFormat = true; }`): [NixOps `examples/trivial-ec2-ebs.nix`](https://github.com/NixOS/nixops-aws/blob/72d0a4c812c6b562ab6a5a18cf98e66193958897/examples/trivial-ec2-ebs.nix#L9-L17)
* `autoFormat` and `autoResize` are defined here: [nixpkgs `nixos/modules/tasks/filesystems.nix`](https://github.com/NixOS/nixpkgs/blob/e4adbfbab8aadf9d80a93d40fb612cb910073af9/nixos/modules/tasks/filesystems.nix#L79()
* The implementation of `autoResize` is here in the stage-1 boot script: [NixOS `stage-1-init.sh`](https://github.com/NixOS/nixpkgs/blob/e4adbfbab8aadf9d80a93d40fb612cb910073af9/nixos/modules/system/boot/stage-1-init.sh#L357)
* The implementation of `autoFormat` is here: [NixOS `filesystems.nix`](https://github.com/NixOS/nixpkgs/blob/e4adbfbab8aadf9d80a93d40fb612cb910073af9/nixos/modules/tasks/filesystems.nix#L274)
  It adds a systemd unit called e.g. `mkfs-dev-xvdb.service` with `after = [ "....device" ]` to run when the device is available.
* Systemd also has a feature for this (but it is not currently  used in NixOS): [`x-systemd.makefs`](https://www.freedesktop.org/software/systemd/man/systemd.mount.html#x-systemd.makefs)
* In both cases `blkid` is used to determine whether there is already a file system on the disk.
* The `fileSystems` declarations are plain NixOS options; as shown above they (and in particular `autoFormat` and `autoResize`) are implemented in NixOS, not nixops.
  How does NixOps know that it should create EBS volumes if the machines are on EC2?
  That happens here: [`nixops_aws/nix/ec2.nix`](https://github.com/NixOS/nixops-aws/blob/72d0a4c812c6b562ab6a5a18cf98e66193958897/nixops_aws/nix/ec2.nix#L516)
  For each of `config.fileSystems` it checks whether it has a `.ec2` attribute.
  Normal NixOS `fileSystems` options don't have that attribute; where is it declared so that it can be set? Here in NixOps: [`fileSystemsOptions.options.ec2 = mkOption`](https://github.com/NixOS/nixops-aws/blob/72d0a4c812c6b562ab6a5a18cf98e66193958897/nixops_aws/nix/ec2.nix#L140-L141), and extending the normal `fileSystems` options [here](https://github.com/NixOS/nixops-aws/blob/72d0a4c812c6b562ab6a5a18cf98e66193958897/nixops_aws/nix/ec2.nix#L476-LL477).


## Behaviour with attached EBS volumes

* When you attach an EBS volume and choose e.g. `/dev/sdb` as `Device name`, it will appear as `/dev/xvdb` on the machine. `/dev/sd*` becomes `/dev/xvd*`.
* When you declare:
  ```nix
  {
    fileSystems."/data" = {
      autoFormat = true;
      fsType = "ext4";
      device = "/dev/xvdb";
    };
  }
  ```
  then formatting + mounting will:
  * **Happen on `nixos-rebuild switch`** if the EBS volume is attached.
    * It outputs: `the following new units were started: data.mount, system-systemd\x2dfsck.slice, systemd-fsck@dev-xvdb.service`
  * **Happen on the next reboot** during which it is attached.
    * You can try this with e.g. `umount /data && wipefs --all /dev/xvdb && reboot`.
  * **Not immediately happen when you attach it** after appling the config, when it wasn't attached during `nixos-rebuild switch`.
    * `journalctl` will show `kernel: blkfront: xvdb: ...` during attachment, but no systemd services units be auto-started.
    * If you want to make formatting / mounting happen after attachment, you may run:
      * `systemctl start data.mount` to format and mount
      * `systemctl status mkfs-dev-xvdb.service` to only format
      You can check the effects with `lsblk` and `blkid`.
  Further:
  * `nixos-rebuild switch` will hang when the volume is not attached, until after around 80 seconds `journalctl` shows `dev-xvdb.device: Job dev-xvdb.device/start timed out.`
    This timeout can be configured with systemd's [`x-systemd.device-timeout=`](https://www.freedesktop.org/software/systemd/man/systemd.mount.html#x-systemd.device-timeout=) option.
    For example, in our NixOS config from above: `options = [ "x-systemd.device-timeout=5s" ];`
    * If you do set a timeout this way, as a side effect `nixos-rebuild` switch will error explicitly instead of ignoring the timeout; I found that a bit surprising and filed it as [nixpkgs #110916: systemd device-timeout settings are not congruent](https://github.com/NixOS/nixpkgs/issues/110916).
    * If you want a timeout not to count as an error (e.g. because the EBS volume may be absent any time), use the `nofail` system mount option (see [here](https://github.com/NixOS/nixpkgs/issues/110916#issuecomment-768161567)).


### Actions upon attaching

If you want the formatting / mounting to happen automatically (and without reboot) as soon as the EBS volume is attached:

* **For auto-formatting only upon attaching**, use
  ```nix
  {
    systemd.services."mkfs-dev-xvdb".wantedBy = [ "dev-xvdb.device" ];
  }
  ```
  As described [here](https://askubuntu.com/questions/1018531/automatically-mount-drive-on-plug-using-systemd), this is sufficient if you know the device path (like `/dev/xvdb`), and if you don't (e.g. want to match it according to some rules), it describes how to do it with `udev`.
* **For auto-mounting upon attaching**, you [need a `udev` rule](https://superuser.com/questions/1364509/how-can-i-start-a-systemd-service-when-a-given-usb-device-ethernet-dongle-is-p):
  ```nix
  {
    services.udev.extraRules = ''
      ACTION=="add", SUBSYSTEM=="block", KERNEL=="xvdb", ENV{SYSTEMD_WANTS}+="data.mount"
    '';
  }
  ```
  If you prefer to think about mount paths instead of `systemd.mount` names, you can also use the handy NixOS function `utils.escapeSystemdPath` to map the `/data` in `fileSystems."/data"` to what systemd calls it in the corresponding `.mount` option, example:
  ```nix
  {
    services.udev.extraRules = ''
      ACTION=="add", SUBSYSTEM=="block", KERNEL=="xvdb", ENV{SYSTEMD_WANTS}+="${utils.escapeSystemdPath "/data"}.mount"
    '';
  }
  ```
  Note it does NOT work to use `systemd.units."data.mount".wantedBy = [ "dev-xvdb.device" ];` or `systemd.units."data.mount".wantedBy = [ "mkfs-dev-xvdb.service" ];`.


## Summary

NixOS already contains all the functionality needed to auto-format and auto-mount block devices upon attachment.

For example, to do it with an AWS EBS volume, you only need:

```nix
{
  fileSystems."/data" = {
    autoFormat = true;
    fsType = "ext4";
    device = "/dev/xvdb";
  };

  services.udev.extraRules = ''
    ACTION=="add", SUBSYSTEM=="block", KERNEL=="xvdb", ENV{SYSTEMD_WANTS}+="${utils.escapeSystemdPath "/data"}.mount"
  '';
}
```
