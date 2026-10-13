# intel-hid-msi

A patched Linux `intel-hid` driver, packaged for DKMS, that makes a short tap on the power button work on MSI laptops whose embedded controller (EC) skips the press event.

Tested on an MSI Pulse GL66 12UEK (EC firmware 1583EMS1.111, BIOS E1583IMS.112) with kernel 7.2.9 and KDE Plasma 6.7.

## The problem

On these laptops the power button reaches the OS through the EC. The DSDT handles EC query `0xCE` by sending `Notify(HIDD, 0xCE)` (press) and query `0xCF` by sending `Notify(HIDD, 0xCF)` (release). The EC only raises `0xCE` once the button has been held for roughly half a second. It raises `0xCF` on every release.

Mainline `intel-hid` maps `0xCE` to `KEY_POWER` and ignores `0xCF` (`KE_IGNORE` in `intel_array_keymap`). A quick tap therefore produces only an ignored release, and no key event reaches userspace. Holding the button a bit longer works.

You can confirm the same behaviour on your machine by enabling ACPI event and EC debug logging and tapping the button:

```sh
echo 0x4 | sudo tee /sys/module/acpi/parameters/debug_layer
echo 'file drivers/acpi/ec.c +p' | sudo tee /sys/kernel/debug/dynamic_debug/control
# tap the power button, then:
sudo dmesg | grep -E 'Query\(0xc[ef]\)|Notify on \[HIDD\]'
# turn it back off:
echo 0x0 | sudo tee /sys/module/acpi/parameters/debug_layer
echo 'file drivers/acpi/ec.c -p' | sudo tee /sys/kernel/debug/dynamic_debug/control
```

A tap that shows `Query(0xcf)` without a preceding `Query(0xce)` is this bug.

## The fix

The driver now remembers whether it has seen `0xCE`. When `0xCF` arrives without a preceding `0xCE`, it reports a full `KEY_POWER` press and release on the 5-button array input device. A normal press (`0xCE` then `0xCF`) behaves exactly as before.

A press seen while the device is in wakeup mode also counts. Otherwise, waking the laptop with the power button would deliver the matching `0xCF` after resume as a lone release and open the logout screen.

The change is in [`intel-hid-msi-short-press.patch`](intel-hid-msi-short-press.patch), against `drivers/platform/x86/intel/hid.c` from Linux v7.2.9. `hid.c` in this repo is that file with the patch applied. Its include of `../dual_accel_detect.h` was changed to `dual_accel_detect.h` so it builds out of tree.

## Install

Requires `dkms` and the headers for each kernel you want it built for. The Makefile passes `LLVM=1` automatically when the kernel was built with clang (CachyOS kernels are).

```sh
sudo cp -r . /usr/src/intel-hid-msi-1.0
sudo dkms install intel-hid-msi/1.0
sudo modprobe -r intel_hid && sudo modprobe intel_hid
modinfo -n intel_hid   # should point into updates/dkms
```

DKMS rebuilds the module on kernel updates. The source is taken from v7.2.9, so on a much newer or older kernel check that the build still succeeds. It also builds against 6.18 LTS.

## Uninstall

```sh
sudo dkms remove intel-hid-msi/1.0 --all
sudo rm -rf /usr/src/intel-hid-msi-1.0
sudo modprobe -r intel_hid && sudo modprobe intel_hid
```

## License

GPL-2.0, same as the kernel source it is derived from. See [LICENSE](LICENSE).
