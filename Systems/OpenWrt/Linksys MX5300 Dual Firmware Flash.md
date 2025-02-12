# Flashing firmware on dual firmware partition devices

## TO DO

The `mtd` command differ from the git commit instructions (see Linksys MX5300 hyperlink above) as I removed the reboot flag so I can flash both partitions. I don't remember if this worked or not. I'll retest this soon and update this section accordingly.

---

I bought a couple of [Linksys MX5300](https://git.openwrt.org/?p=openwrt/openwrt.git;a=commit;h=70fd815e57dc98cef88828faa5ce71717540e99e) from Amazon renewed with no intention of keeping the default Linksys firmware. However I wanted to make sure both firmware partitions were totally flashed before usage.

I don't plan on immediately deploying the devices so I wanted to flash the firmware on both partitions but also without performing the first boot procedure on either of the images.

---

1. You'll need to replace `kernel` and `alt_kernel` with the names of the kernal A/B partition names of your device.
2. You'll need to replace `openwrt.bin` with the name of your firmware image. I will not cover copying the firmware image to your device.
3. This assumes that the booted partition is the first one (`kernel`/`boot_part` = `1`). If you're booted to `alt_kernel` or `boot_part` = `2` already, simply swap those in the `fw_setenv` command.

```ash
mtd -e kernel -n write openwrt.bin kernel
mtd -e alt_kernel -n write openwrt.bin alt_kernel
fw_setenv boot_part 2 && firstboot && reboot
fw_printenv boot_part
```
