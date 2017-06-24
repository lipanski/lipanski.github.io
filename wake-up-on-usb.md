## How to wake up your Linux machine on USB input

Use the `lsusb` command to figure out the vendor and product ID of the USB device you are interested in:

```sh
lsusb
```

This will print out something like:

```
Bus 001 Device 009: ID 045e:07b2 Microsoft Corp. 
Bus 001 Device 010: ID 046d:c52b Logitech, Inc. Unifying Receiver
```

The vendor and product IDs are the two hexadecimal values separated by a colon.

In my case, I'm interested in the Logitech keyboard, for which the vendor ID would be `046d` and the product ID would be `c52b`.

The next step is to find the directory under `/sys/bus/usb/devices` that matches the product:

```sh
tail /sys/bus/usb/devices/*/idVendor
tail /sys/bus/usb/devices/*/idProduct
```

This will print out the contents of every `idVendor` and `idProduct` file in the `/sys/bus/usb/devices` subdirectories, including the full path of the file that contains this.

Based on the output of these commands, identify the subdirectory matching your product. In my case, it was `/sys/bus/usb/devices/1-2.1`.

The last step is to **enable waking up from stand-by**. This is as easy as:

```sh
echo "enabled" > /sys/bus/usb/devices/1-2.1/power/wakeup
```

If you ever want to **disable waking up from stand-by**, replace `enabled` with `disabled`:

```sh
echo "disabled" > /sys/bus/usb/devices/1-2.1/power/wakeup
```
