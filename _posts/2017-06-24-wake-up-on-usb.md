---
layout: default
title: "Linux: How to wake up from stand-by on USB input"
tags: linux
---

## Linux: How to wake up from stand-by on USB input

Use the `lsusb` command to figure out the vendor and product ID of the USB device you are interested in:

```sh
lsusb
```

This will print out something like:

```
Bus 001 Device 009: ID 045e:07b2 Microsoft Corp.
Bus 001 Device 010: ID 046d:c52b Logitech, Inc. Unifying Receiver
```

The vendor and product IDs are the two hexadecimal values separated by a colon - like `046d:c52b`.

In my case, I'm interested in the Logitech keyboard, for which the vendor ID would be `046d` and the product ID would be `c52b`.

The next step is to find the directory under `/sys/bus/usb/devices` that matches the product. Have a look inside this directory:

```sh
ls /sys/bus/usb/devices
```

You will find a list of devices that don't seem to be correlated with the `lsusb` output. The correlation is established by the contents of the `idVendor` and `idProduct` files inside these subdirectories. You can use `tail` with a wildcard to output the contents of these files, including their exact path, and identify the subdirectory of interest:

```sh
# Replace 046d with your vendor ID
tail /sys/bus/usb/devices/*/idVendor | grep -B1 046d

# Replace c52b with your product ID
tail /sys/bus/usb/devices/*/idProduct | grep -B1 c52b
```

Look for the subdirectory name where **both** the vendor and the product ID match. In my case, it was `/sys/bus/usb/devices/1-2.1`.

The final step is to **enable waking up from stand-by**:

```sh
# Replace 1-2.1 with the subdirectory name matching your product
echo "enabled" > /sys/bus/usb/devices/1-2.1/power/wakeup
```

In case you ever want to **disable waking up from stand-by**, replace `enabled` with `disabled`:

```sh
# Replace 1-2.1 with the subdirectory name matching your product
echo "disabled" > /sys/bus/usb/devices/1-2.1/power/wakeup
```
