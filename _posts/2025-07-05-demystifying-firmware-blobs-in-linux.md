---
title: "Demystifying Firmware Blobs in Linux"
categories: [linux-kernel]
tags: [firmware, internals]
---

Most Linux systems rely on binary blobs to activate the full capabilities of hardware like Bluetooth and Wi-Fi chips. But what are these blobs, and how does the kernel load them?
### Understanding Firmware Blobs
The Linux kernel supports loading files (blobs) from userspace to enable specific hardware functionalities. These files may include **CPU microcode**, **firmware for device microcontrollers**, or **auxiliary data used by drivers**. Some of this data is optional, and only used to add additional features of the target device.
>The main idea is to load the blob in the kernel and upload it to the target device (like a Bluetooth chip from MediaTek, for example). We'll dive into this later, but for now, let's move on.
{: .prompt-tip }

You can find the blobs provided by vendors [here](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree). Usually this repository (or one of its mirrors) is the upstream source for `linux-firmware` package in distributions, unless the distribution decides to compress each blob to reduce size.

One important thing to note about these blobs is that they're **closed-source**. Some of them are raw data, and some of them are ELF or PE files. In any case, most (if not all) of them include license files (like `LICENSE.<vendor>`) explicitly **prohibiting reverse engineering**. For example, here's a line from `LICENSE.amd-ucode`, which is for AMD CPU microcode:
> You may not reverse engineer, decompile, or disassemble this Software or any portion thereof.

[Linux libre](https://www.fsfla.org/ikiwiki/selibre/linux-libre/) removed blobs since they are non-free software after all. It's worth mentioning that since the blob is uploaded to the target device and not executed as part of the kernel itself, its licensing is separate and doesn't conflict with the GPLv2, which is the main license of the Linux kernel.
### Firmware Lookup Directories
By default, the `firmware_class` module looks for firmware files in the following directories: 
```
- /lib/firmware/updates/UTS_RELEASE/
- /lib/firmware/updates/
- /lib/firmware/UTS_RELEASE/
- /lib/firmware/
```
On most common GNU/Linux distributions, the contents of the `linux-firmware` package are typically found in `/lib/firmware`, and the files are usually stored in a compressed format. Adding a custom path is also easy:
```bash
echo $CUSTOMIZED_PATH | sudo tee /sys/module/firmware_class/parameters/path
# OR using firmware_class.path=$CUSTOMIZED_PATH
```
### Firmware Loading Flow in the Linux Kernel
The firmware loader of the kernel is pretty straightforward. I'm going to use the source code of [`btmtk`](https://elixir.bootlin.com/linux/v6.15.4/source/drivers/bluetooth/btmtk.c) driver as an example.
#### Requesting the Firmware
The MediaTek **MT7922** Bluetooth chip needs several firmware blobs, like this one:
```
mediatek/BT_RAM_CODE_MT7922_1_1_hdr.bin.zst
```
The driver loads it using [`request_firmware()`](https://elixir.bootlin.com/linux/v6.15.4/source/drivers/base/firmware_loader/main.c#L962):
```c
err = request_firmware(&fw, fwname, &hdev->dev);
```
Even though the file is compressed (`.zst`), you just pass the base name (`.bin`). The kernel tries known suffixes like `.zst` or `.xz` if compression support is enabled by setting `CONFIG_FW_LOADER_COMPRESS_<ZSTD OR XZ>`:
```c
_request_firmware() {
	// ...
#ifdef CONFIG_FW_LOADER_COMPRESS_ZSTD
	if (ret == -ENOENT && nondirect)
		ret = fw_get_filesystem_firmware(device, fw->priv, ".zst",
						 fw_decompress_zstd); // <--- HERE
#endif
#ifdef CONFIG_FW_LOADER_COMPRESS_XZ
	if (ret == -ENOENT && nondirect)
		ret = fw_get_filesystem_firmware(device, fw->priv, ".xz", fw_decompress_xz);
#endif
	// ...
}
```
During early boot, kernel calls two functions to load the blobs:
1. [`wait_for_initramfs()`](https://elixir.bootlin.com/linux/v6.15.4/source/init/initramfs.c#L764)
2. [`kernel_read_file_from_path_initns()`](https://elixir.bootlin.com/linux/v6.15.4/source/fs/kernel_read_file.c#L147)
This means it reads the firmware **from the init process's filesystem view** (typically the *initramfs*). Initramfs is the temporary root filesystem loaded during early boot before the real root filesystem is mounted. So unless you build the firmware into the kernel image (`CONFIG_EXTRA_FIRMWARE`), **it has to be available in the initramfs**.

>Compressed blobs don't work with `CONFIG_EXTRA_FIRMWARE`. For built-in drivers (like CPU microcode), firmware **must be uncompressed**. **MT7922** is also one of those drivers.
{: .prompt-warning }

Later, if the module loads after userspace is up, tools like `udev` can supply the firmware via `/sys/class/firmware`.
#### Sending Firmware to the Device
Once `request_firmware` succeeds, it fills in two fields for us:
- `fw->data`: a pointer to the firmware content in memory
- `fw->size`: the size of the firmware blob
Now, all that's left is to send the firmware to the device. Our approach to do this depends on the hardware interface. This could be over USB, SPI, or any other transport protocols. In the case of the `btmtk`, the firmware is sent over USB using MediaTek's WMT (Wireless Management Transport) protocol.
The driver breaks the firmware into 250-byte chunks and sends each chunk using `wmt_cmd_sync()`:
```c
while (dl_size > 0) {
	dlen = min_t(int, 250, dl_size);
	// ...
	wmt_params.flag = flag;
	wmt_params.dlen = dlen;
	wmt_params.data = fw_ptr;

	err = wmt_cmd_sync(hdev, &wmt_params); // <-- HERE
	// [ error-handling shenanigans ]
	dl_size -= dlen;
	fw_ptr += dlen;
}
```
`wmt_cmd_sync` is a function pointer that in here, points to [`btmtk_usb_hci_wmt_sync()`](https://elixir.bootlin.com/linux/v6.15.4/source/drivers/bluetooth/btmtk.c#L580). Inside that function, the driver sends the command using a vendor-specific HCI opcode: 
```c
err = __hci_cmd_send(hdev, 0xfc6f, hlen, wc);
```
`wc` is the WMT command buffer that includes the firmware chunk.

After sending, the driver subits a USB URB to wait for the chip's response:
```c
err = btmtk_usb_submit_wmt_recv_urb(hdev);
```
The driver then waits for confirmation from the device that the chunk was received and processed correctly before continuing to the next chunk. 
### Further Reading
- [Firmware Documentation in the Linux Kernel](https://www.kernel.org/doc/html/latest/driver-api/firmware/index.html)
- [Gentoo Wiki - Linux Firmware](https://wiki.gentoo.org/wiki/Linux_firmware)
- [Linux-libre Project](https://www.fsfla.org/ikiwiki/selibre/linux-libre)
