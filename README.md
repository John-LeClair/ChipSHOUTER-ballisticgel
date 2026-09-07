# EMFI-TARGET

EMFI-TARGET is an Electro-Magnetic Fault Injection (EMFI) target. It is specially designed to help you understand fault injection patterns for a given tip.

It uses a large SRAM chip as a target, which has a relatively simple layout. This lets you understand how much of a given chip you are corrupting. EMFI-TARGET is based on NewAE's incredibly useful ChipSHOUTER® CW521 Ballistic Gel. I found NewAE's CW521 Ballistic Gel constantly out of stock - thus I made  manufactured my own. Manufactured in the United States of America with globally sourced parts. 

![](cw520_photo.jpg)

## GIT Layout ##

The GIT repository contains the following:

1) Kicad PCB files. 
2) Firmware for the microcontroller.
3) Python library / PC application.

# Virtual Development Environment Setup (Linux)

Creating a pyenv environment (ie: Python 3.10.12).

```bash
cd ~/venvs
python3 -m venv emfi-blaster
source ~/venvs/emfi-blaster/bin/activate
```

## Install required libraries

```bash
pip install numpy
pip install matplotlib
pip install libusb1
pip install debugpy   # optional
```

## Install ChipWhisperer repo

```bash
cd ~/dev/
git clone https://github.com/John-LeClair/ChipSHOUTER-ballisticgel.git

cd ~/dev
git clone https://github.com/newaetech/chipwhisperer.git
cd chipwhisperer
python -m pip install -e .
```



## PC Application ##

The PC application is a simple example of using the Python library. This application does the following (via the library)

1. Downloads a pattern to the SRAM chip.
2. Waits for fault injection.
3. Uploads SRAM chip contents & determines corrupt locations.
4. Graphs map of physical SRAM locations (NB: not yet fully working).

The SRAM pattern can be something besides the random pattern, but the random pattern ensures "odd" corruptions (such as shorting address lines etc) will easily be caught.

An example of using the file is given at the end of ballisticgel.py, see the following:

    emfi_target = EMFI_TARGET()
    emfi_target.con()
    
    doplot = False
    savefile = None
    #savefile = 'error_locations.bin' 
    
    #Raw method recommended - same speed and more flexible
    use_raw_method = True

    while True:
        try:        
            if use_raw_method:
                print "Writing data..."
                emfi_target.raw_test_setup()
                raw_input("Hit enter when glitch inserted")
                results = emfi_target.raw_test_compare()
            else:
                print "Writing data..."
                emfi_target.seed_test_setup()
                raw_input("Hit enter when glitch inserted")
                results = emfi_target.seed_test_compare()
            
            errdatay = results['errdatay']
            errdatax = results['errdatax']
            errorlist = results['errorlist']
            
            if doplot:
                plt.plot(errdatax, errdatay, '.r')
                plt.axis([0, 8192, 0, 4096])
                plt.show()

            if savefile:
                with open(savefile, "wb") as errfile:
                    errfile.write(bytearray(errorlist))
        except:
            emfi_target.close()

The "graph" that pops up afterwards is slightly bogus - the physical map of the SRAM is not yet accurate. But the most interesting aspect is that you can see number of bit flips (positive/negative), and total number of bytes corrupted.

### Result Format ###

The result information is provided in a dictionary. Depending if you use the fast (but less detailed) method or the slow (but more detailed) method you may not have all of these fields. It currently provides you with this:

 - 'errorlist': A list of addresses of each byte error. The length of this is the number of byte errors.
 - 'errdatax', 'errdatay': errdatax & errdatay attempt to provide a map of locations on SRAM chip where errors occurred. Until mapping is complete this is not fully accurate.
 - 'set_errors': Number of bit-set errors that occurred. Note number of bit errors is different from number of byte errors.
 - 'reset_errors': Number of bit-reset errors that occurred. Note number of bit errors is different from number of byte errors.

## Building Firmware ##

The firmware uses the ChipWhisperer capture build system (naeusb). Navigate to
`firmware/emfi_target`
run `make`.

The firmware build requires `make` and `arm-none-eabi-gcc`.

## Drivers ##

As of commit `f62ccdf0ea2d611deabf48ec3ad5db759205dbb0` and firmware version 2.0.0, 
the EMFI-TARGET now uses the same WCID driver assignment as ChipWhisperer devices,
meaning no custom drivers need to be installed.

If you have old firmware/drivers and want to update, the easiest method is to:

1. Plug your EMFI-TARGET board in
1. Short JP1 (labelled ERASE)
1. Unplug + replug your EMFI-TARGET
1. Run the following code:

```python
from ballisticgel import program_sam_firmware
program_sam_firmware()
```

Note that the same `upgrade_firmware()` method is now available on the CW521 object:

```python
from ballisticgel import EMFI_TARGET
emfi_target = EMFI_TARGET()
emfi_target.upgrade_firmware()
```

## Flashing Firmware via Bossac
# EMFI-TARGET Firmware Build & Flash Guide

This guide outlines the complete step-by-step process for configuring, compiling, and flashing custom firmware to the NCondor Embedded Technology, LLC  EMFI-TARGET platform natively via Linux without Python.

---

## 🛠️ Step 1: Install Build Dependencies & Flash Tools

Before compiling or flashing, install the standard GNU compilation stack, the ARM embedded toolchain, and the native BOSSA toolchain utility (`bossac`).

```bash
sudo apt update
sudo apt install build-essential git gcc-arm-none-eabi bossa-cli
```

---

## ⚙️ Step 2: Compile the Application

Wipe existing artifact objects and trigger the project Makefile compiler sequence from inside the `firmware/emfi_target` directory:

```bash
make clean
make
```
*This step produces the flashable native firmware execution payload at: `emfi_target.bin`*

---

## 🔄 Step 5: Force the EMFI-TARGET into Bootloader Mode

To make the board appear as a flashable interface to your Linux system, you must clear the existing flash memory to expose the ROM-resident SAM-BA bootloader.

1. Locate the two small physical buttons on the top edge of the red PCB.
   * **Left Button (`nRST`):** Master hardware reset.
   * **Right Button (`ERASE`):** Memory array clear line.
2. While the board is connected and powered via USB, **press and hold the Erase pins or button for 1–2 seconds**. 
3. This completely clears out the corrupt or active application stack so the microchip defaults straight to its integrated hardware boot protocol.

---

## ⚡ Exact Flashing Sequence (Step-by-Step)

Because there isn't a third dedicated boot-selection key, you must execute a strict physical dual-button sequence to isolate the chip and expose its flash controller interfaces cleanly over USB.

### 🏃‍♂️ Execution Sequence
1. **Connect** the EMFI-TARGET to your Linux computer via the USB cable.
2. **Press and hold** down the **Erase (Right)** button.
3. While keeping the Erase button held down, **tap** the **Reset (Left)** button once.
4. Continue holding the **Erase (Right)** button for 1–2 additional seconds, then **release** it.
5. **Disconnect** the USB cable from the board, then **reconnect** it.

### 📝 Final Payload Write Execution
Verify that the system heartbeat LED remains completely unlit (indicating safe bootloader execution mode). Locate the newly assigned virtual serial interface profile via `ls /dev/ttyACM*`, then run the raw `bossac` terminal utility command:

```bash
bossac --port=ttyACM0 -e -w -v -b emfi_target.bin
```


## Legal ##
EMFI-TARGET is based on the Ballistic Gel open-source project which is GPL licensed - thus EMFI-TARGET is released with the GPL license.

The EMFI-TARGET uses Condor Embedded Technology, LLC's USB VID/PID. Microchip's USB-IF license disallows sub-licensing. Therefore, if selling your own verison of this hardware, you must obtain your own VID/PID. 

The EMFI-TARGET can be purchased by contacting jleclair@condorembeddedtech.com. 

Ballistic Gel is part of the ChipSHOUTER project (which is itself related to the ChipWhisperer project). It is also known as the CW521 target board.

Ballistic Gel is an open-source project, and is released with the GPL license. Assembled boards can be purchased from NewAE Technology Inc at https://store.newae.com .

ChipSHOUTER is a registered trademark of NewAE Technoloy Inc. Note you CANNOT sell boards using the ChipSHOUTER name without permission, and you cannot use NewAE Technology Inc's USB VID on your own products as the USB-IF license disallows sub-licensing in this manner. If you change the VID/PID, simply change the associated VID/PID in the .inf (driver) file as needed.

