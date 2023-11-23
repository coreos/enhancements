# z/VM Guest Secure IPL (Secure Boot-like protection)
---

# [Overview](https://www.ibm.com/docs/en/zvm/7.3?topic=ipl-guest-secure)

z/VM® supports guest secure IPL. Guest secure IPL supports the NIAP (National Information Assurance Partnership) operating system protection profile, which supports the Common Criteria certification.

A z/VM user can request that the machine loader validate the signed IPL code by using the security keys that were previously loaded by the customer into the HMC certificate store. The validation ensures that the IPL code is intact, unaltered, and originates from a trusted build-time source.

This support provides the ability for a Linux guest to exploit hardware to validate the code being booted, helping to ensure it is signed by the client or its supplier.

Support is provided for the following device types:
- SCSI devices.
- ECKD devices.

---

# Enabling Secure IPL at installation time (day-1)

Assuming zVM is ready for secure boot, we can setup LOADDEV at installation time

## coreos-installer

1) Add new `coreos.inst.secure_ipl` karg and `--secure-ipl` option. `coreos-installer-generator` appends the switch when karg is provided
2) During installation we check for `--secure-ipl` and use `vmcp` tool to set LOADDEV
3) Add new systemd unit `coreos-installer-reboot-loaddev.service` to restart from LOADDEV (which immediately terminates running CoreOS VM)

# Enabling Secure IPL on existing system (day-2)

This is for informational purposes only. We expect users to only turn on Guest Secure IPL at installation time.

1) Login into RedHat CoreOS and ensure the output:
    ```
    $ cat /sys/firmware/ipl/has_secure
    1
    ```
2) Set target disk as zVM LOADEVICE
- ECKD

    Assuming RHCOS is installed on DASD disk `0.0.5223`, from zVM terminal execute:
    ```
    # cp set loaddev eckd dev 5223 secure
    ```
- SCSI

    Assuming RHCOS is installed on FCP disk `0.0.8007,0x500507630400d1e3,0x4001404c00000000`, from zVM execute:
    ```
    # cp set loaddev dev 8007 portname 50050763 0400d1e3 lun 4001404c 00000000 secure
    ```
3) Update bootloader
    ```
    $ sudo unshare --mount bash -c 'mount -o remount,rw /boot && zipl -V --secure 1'
    ```
4) Poweroff RHCOS and start it from zVM terminal:
    ```
    # cp ipl loaddev
    ```

# Ensure system runs with Secure IPL

Preferred way to check whether Secure IPL is used:
```
$ cat /sys/firmware/ipl/secure
1
```
But currently this functionality could be broken on some kernel versions.

Another way to check is:
```
$ dmesg | grep "Secure-IPL enabled"
[    0.029829] setup: Linux is running with Secure-IPL enabled
```
