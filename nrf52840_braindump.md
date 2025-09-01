# Some notes on the NRF52840

## Flashing your NRF52840 based device
There's a few options for flashing:  
- Use UF2 mode and copy a UF2 file onto the drive that is presented. Recommended!
- Use serial DFU mode (over USB connection) with a zip file and the adafruit-nrfutil program
- Use OTA DFU mode with a zip file and an app on your phone (DFU/nRF Connect/etc)
- Use a SWD or J-Link device and associated software (OpenOCD/pyocd/etc)

### UF2 DFU mode
This is the recommended option, as it is the safest and most straightforward. Generally you only need to double-press the reset button within 0.5 seconds and the device will show up in your computer as a disk drive. Then you copy a UF2 file to the drive to flash it. You can flash firmware or bootloader this way, but not SoftDevice afaik.

### Serial DFU mode
This one is pretty simple as well, you just need to know what serial port the device is on and then you can run the following:  
``adafruit-nrfutil dfu serial -sb -t 1200 -p PORTNUM -pkg package-to-flash.zip``  
e.g. ``adafruit-nrfutil dfu serial -sb -t 1200 -p COM46 -pkg TrackerL1-bootloader.zip``  

``dfu serial`` means we want to do a Device Firmware Update using a serial connection.  
``-sb`` means the device only has a single bank to store firmware  
``-t 1200`` means we want to open and close the COM port with 1200 baud rate in order to trigger the device to go into DFU mode  

This mode allows you to flash firmware and bootloader zip packages. Bootloader zip packages are also able to set certain (all?) UICR registers when flashed this way so that's a bonus!

### OTA DFU mode
This one is slightly less simple, and has some risk associated with it. The risk is that the OTA update fails and you're left with a device that needs updating by USB cable anyway, so not a big deal unless your device is not easily accessible. This uses the DFU/nRF Connect/etc app on your phone or tablet. Might be possible from Windows/Mac/Linux as well but I haven't tried.

### SWD with STLinkV2 and pyocd.

When flashing a .bin file you must take care to set the correct offset for where in flash the file should be written. When flashing a .hex file you don't need to worry about offsets, as the .hex file specifies the offsets for you.

// Write a whole flashdump bin file  
``pyocd flash -t nrf52840 -a 0x0 flashdump.bin``

// Write a bootloader bin dump file (note that if you've done a full flash erase you will also need to write the MBR and SoftDevice)  
``pyocd flash -t nrf52840 -a 0xF4000 bootloader.bin``

// Write a hex file  
``pyocd flash -t nrf52840 dump.hex``


--- 

### Creating a UF2 file from a .hex or .bin file.
Use the uf2conv.py python script. If you use a bin file you must specify the correct offset for your device.

``python uf2conv.py firmware.bin -c -f 0xADA52840 -b APPLICATION_START_OFFSET -o firmware.uf2`` // APPLICATION_START_OFFSET is dependant on your softdevice version, see below for some offsets.

### Creating a firmware UF2 file from a firmware hex file
``python uf2conv.py firmware.hex -c -f 0xADA52840 -o firmware.uf2`` // No need for offsets, hex files specify them internally.

### Creating a bootloader UF2 file from a hex file
When creating a bootloader UF2 file you need to specify a special family ID so that the bootloader knows it will need to update itself. If hand crafting a hex file (say from a flash dump), there's a few gotchas that I'll detail later.

``python uf2conv.py bootloader.hex -f 0xd663823c -c -o bootloader.uf2``

---

### How to dump the flash from NRF52840 over SWD.
I did it using pyocd, here's the steps.  

This will dump the entire 1Mb flash to a .bin file. This includes everything, except the UICR registers. It might be possible to dump them in this way as well but I haven't tried yet.  
``pyocd cmd -c 'savemem 0x0 1048576 flashdump.bin'``

Here's a few other command examples to dump specific areas of flash.  
``savemem 0xF4000 49152 bootloader.bin`` // dump the bootloader to file, note you will need more than just a bootloader to make a device boot.  
``savemem 0x26000 815104 firmware.bin`` // dump the application flash from a device running SoftDevice S140 v6.1.1  
``savemem 0x27000 811008 firmware.bin`` // dump the application flash from a device running SoftDevice S140 v7.3.0  
``savemem 0xED000 28672 userdata.bin`` // dump the UserData area



---
### Creating hex files from bin files
Use bin2hex.py:  
``bin2hex.py --offset ZZZZZZ infile.bin outfile.hex``

Examples:  
// Convert entire flashdump.bin to hex. This is a good start for working with your flash dump.  
``bin2hex.py --offset 0x0 flashdump.bin flashdump.hex``  

---

### Creating a bootloader .zip package and UF2 from a flash dump

Coming soon if I can be bothered.

---

### Some useful hex files and where to find them in the Adafruit nRF52 Bootloader source tree.
These files are useful if you're handcrafting a hex file based on a dump. For example for recreating a bootloader zip package from a dump.

lib/softdevice/mbr/hex/mbr_nrf52_2.4.1_mbr.hex  
lib/softdevice/s140_nrf52_7.3.0/s140_nrf52_7.3.0_softdevice.hex  // softdevice includes the MBR  
lib/softdevice/s140_nrf52_6.1.1/s140_nrf52_6.1.1_softdevice.hex  // softdevice includes the MBR  

---

### NRF52840 flash offsets in no particular order

0x0 - MBR (from what I've seen it's always the same, you can find it in the Adafruit nRF52 Bootloader source files)  
0x1000 - SoftDevice (length varies depending on SoftDevice version, s140 6.1.1 finishes at 0x26000, s140 7.3.0 finishes at 0x27000)  
0x26000 - Application start (SoftDevice-s140-6.1.1)  
0x27000 - Application start (SoftDevice-s140-7.3.0)  
0xF4000 - Bootloader  
0xFD800 - Bootloader settings  
0xFE000 - MBR params (not used by adafruit bootloader?)  

Offsets starting at 0x10001000 refer to the UICR registers (User Information Configuration Registers). These can be set when flashing via a debugger with a hex file, or directly using debugger commands, or some (all?) can also be set by a zip package (bootloader package only?) when flashing using adafruit-nrfutil.

Some examples of UICR offsets  
0x10001014 - UICR.BOOTLOADERADDR (tells MBR where the bootloader lives)  
0x10001018 - UICR.MBRPARAMADDR (tells MBR where MBR params live)  
0x1000120C - UICR.NFCPINS (tells the chip whether p0.09/p0.10 should be used for NFC antenna or as GPIO)  
0x10001304 - UICR.REGOUT0 (sets the output voltage of the onchip regulator)  

---

### Fixing UICR.REGOUT0

If you do certain types of erase (mass erase or chip erase maybe?) you can end up with an unbootable device due to wiping the UICR.REGOUT0 register which is used to set the output voltage of the on-chip voltage regulator. Flashing a bootloader doesn't usually set this register, although it's worth looking into whether this might be possible to include in firmware.zip packages.

To set UICR.REGOUT0 voltage to 3.3v:  
Using pyocd and gdb:  
```
pyocd gdb
set *0x10001304 = 5
```

Using nrfjprog: // NOT TESTED!  
``nrfjprog --memwr 0x10001304 --val 0xFFFFFFFD`` // NOT TESTED!  

---

### Notes on NRF52840 boot process as I (mostly) understand it.

The NRF52840 begins execution with the MBR at 0x0, which checks UICR.BOOTLOADERADDR for the bootloader address and passes to there. The bootloader runs some logic to check whether to initiate a DFU. If not then execution passes to the application at an address found in the bootloader settings area.

I might expand on this at some point.


---

### Notes on the .hex file format
Formally known as Intel Hex, this format was created by Intel in 1973 and originally intended for use with paper tape (!!).  
Turns out it's still in pretty widespread usage today for programming flash ROMs and microcontrollers

I find it much nicer than .bin to work with, since the offsets for the data are contained in the file itself and since it's an ASCII based format you can learn to understand it and then edit it yourself with just a simple text editor.