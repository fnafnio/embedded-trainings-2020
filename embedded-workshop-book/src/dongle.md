# nRF52840 Dongle

Next, we'll look into the radio API exposed by the `dk` HAL. But before that we'll need to set up the nRF52840 Dongle.

From this section on, we'll use the nRF52840 Dongle in addition to the nRF52840 DK. We'll run some pre-compiled programs on the Dongle and write programs for the DK that will interact with the Dongle over a radio link.

 **💬 How to find the buttons on the Dongle:**
 Put the Dongle in front of you, so that the side with the parts mounted on faces up. Rotate it, so that the narrower part of the board, the surface USB connector, faces away from you.
 The Dongle has two buttons. They are next to each other in the lower left corner of the Dongle. The reset button (RESET) is mounted sideways, it's square shaped button faces you. Further away from you is the round-ish user button (SW1), which faces up.

The Dongle does not contain an on-board debugger, like the DK, so we cannot use `probe-rs` tools to write programs into it. Instead, the Dongle's stock firmware comes with a *bootloader*.

When put in bootloader mode the Dongle will run a bootloader program instead of the last application that was flashed into it. This bootloader program will make the Dongle show up as a USB CDC ACM device (AKA Serial over USB device) that accepts new application images over this interface. We'll use the `nrfutil` tool to communicate with the bootloader-mode Dongle and flash new images into it.

✅ Connect the Dongle to your computer. Put the Dongle in bootloader mode by  pressing its *reset* button.

When the Dongle is in bootloader mode its red LED will oscillate in intensity. The Dongle will also appear as a USB CDC ACM device with vendor ID `0x1915` and product ID `0x521f`.

You can also use our `usb-list` tool, a minimal cross-platform version of the `lsusb` tool, to check out the status of the Dongle.

✅ Run `usb-list` to list all USB devices; the Dongle will be highlighted in the output, along with a note if in bootloader mode.

Output should look like this:
``` console
$ usb-list
(..)
Bus 001 Device 016: ID 1915:521f <- nRF52840 Dongle (in bootloader mode)
```

Now that the device is in bootloader mode browse to the `boards/dongle` directory. You'll find some `*.hex` files there. These are pre-compiled Rust programs that have been converted into the Intel Hex format that the `nrfutil` tool expects.

For the next section you'll need to flash the `loopback.hex` file into the Dongle. There are two ways to do this. You can make 2 long `nrfutil` invocations or you can use our `dongle-flash` tool, which will invoke `nrfutil` for you. The `dongle-flash` way is shown below:

✅ Run the following command:

``` console
$ dongle-flash loopback.hex
```

Expected output:
``` console
packaging iHex using nrfutil ...
DONE
  [####################################]  100%
Device programmed.
```

After the device has been programmed it will automatically reset and start running the new application.

The `loopback` application will *blink* the red LED in a heartbeat fashion: two fast blinks (LED on then off) followed by two periods of silence (LED off). The application will also make the Dongle enumerate itself as a CDC ACM device.

✅ Run the `usb-list` tool to see the newly enumerated Dongle in the output:

``` console
$ usb-list
(..)
Bus 001 Device 020: ID 2020:0309 <- nRF52840 Dongle (loopback.hex)
```

The `loopback` app will log messages over the USB interface. To display these messages on the host we have provided a cross-platform tool: `serial-term`.

❗ Do not use serial terminal emulators like `minicom` or `screen`. They use the USB TTY ACM interface in a slightly different manner and may result in data loss.

✅ Run the `serial-term` application. You should see the following output:

``` console
$ serial-term
deviceid=588c06af0877c8f2 channel=20 TxPower=+8dBm app=loopback.hex
```

This line is printed by the `loopback` app on boot. It contains the device ID of the dongle, a 64-bit unique identifier (so everyone will see a different number); the radio channel that the device will use to communicate; and the transmission power of the radio in dBm.

If you don't get any output from `serial-term` check [the USB dongle troubleshooting section][usb-issues].

[usb-issues]: /troubleshoot-usb-dongle.html

## Interference

At this point you should *not* get more output from `serial-term`.

❗If you get "received N bytes" lines in output like this:

``` console
$ serial-term
deviceid=588c06af0877c8f2 channel=20 TxPower=+8dBm
received 7 bytes (CRC=Ok(0x2459), LQI=0)
received 5 bytes (CRC=Ok(0xdad9), LQI=0)
received 6 bytes (CRC=Ok(0x72bb), LQI=0)
```

That means the device is observing interference traffic, likely from 2.4 GHz WiFi or Bluetooth. In this scenario you should switch the listening channel to one where you don't observe interference. Use the `tools/change-channel` tool to do this. The tool takes a single argument: the new listening channel which must be in the range 11-26.

``` console
$ change-channel 11
requested channel change to channel 11
```

Then you should see new output from `serial-term`:

``` console
deviceid=588c06af0877c8f2 channel=20 TxPower=+8dBm
(..)
now listening on channel 11
```

Leave the Dongle connected and the `serial-term` application running. Now we'll switch back to the Development Kit.
