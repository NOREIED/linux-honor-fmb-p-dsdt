# linux-honor-fmb-p-dsdt
fix fmb-p on linux issues

## tested info
bios version: 1.13

OS: fedora 44 (kernel 6.19)
##
on the first, my english is bad,sorry...

i don't have global version machine,so i can't fix and test global version. you can read this article and fix by yourself.

even the touchscreen is fixed by dsdt,but you still need to block a broken keyboard interface


## fix DSDT

install acpica-tools and extract dsdt

**use no dsdt patch linux to run `acpidump -b`, don't use dsdt patched linux or windows!!!**

```
# Fedora
sudo dnf install acpica-tools
# Debian/Ubuntu
sudo apt install acpica-tools
```

```
acpidump -b
iasl -e ssdt*.dat -d dsdt.dat
```
after completing this step, you can see some errors

just find and remove one of `ssdt*.dat` with error object

you can use this command:`grep -rn "HS03" .`

on my device, it's ssdt23.dat(maybe different)

after remove the incorrect ssdt,decompile again

`iasl -e ssdt{1..22}.dat ssdt{24..26}.dat -d dsdt.dat`

open dsdt.dsl and search `NFC0`

just like this

```
Device (NFC0)
            {
                Name (_ADR, Zero)  // _ADR: Address
                Name (_HID, "NTAG0001")  // _HID: Hardware ID
                Name (SBFU, Buffer (0x23)
                {
                    /* 0000 */  0x8E, 0x1E, 0x00, 0x02, 0x00, 0x01, 0x02, 0x00,  // ........
                    /* 0008 */  0x00, 0x01, 0x06, 0x00, 0x80, 0x1A, 0x06, 0x00,  // ........
                    /* 0010 */  0x57, 0x00, 0x5C, 0x5F, 0x53, 0x42, 0x2E, 0x50,  // W.\_SB.P
                    /* 0018 */  0x43, 0x30, 0x30, 0x2E, 0x49, 0x32, 0x43, 0x31,  // C00.I2C1
                    /* 0020 */  0x00, 0x79, 0x00                                 // .y.
                })
                Name (SBGF, Buffer (0x25)
                {
                    /* 0000 */  0x8C, 0x20, 0x00, 0x01, 0x00, 0x01, 0x00, 0x01,  // . ......
                    /* 0008 */  0x00, 0x00, 0x00, 
```

find this line:`INT1 = GNUM (0x0014080A)`

replace this code with
```
Method (_INI, 0, NotSerialized)
                {
                    INT1 = GNUM (0x0014080A)
                }
```

now, your file is seemd like this

```
                    /* 0018 */  0x00, 0x5C, 0x5F, 0x53, 0x42, 0x2E, 0x47, 0x50,  // .\_SB.GP
                    /* 0020 */  0x49, 0x30, 0x00, 0x79, 0x00                     // I0.y.
                })
                CreateWordField (SBGF, 0x17, INT1)
                Method (_INI, 0, NotSerialized)
                {
                    INT1 = GNUM (0x0014080A)
                }
                Method (_STA, 0, NotSerialized)  // _STA: Status
                {
                    If (((TPDT == One) || (TPDT == 0x02)))
                    {
                        Return (0x0F)
                    }
                    Else
                    {
                        Return (Zero)
                    }
```

after this,search`FTSC1000`

turn up and find this line `CreateDWordField (SBFI, 0x05, INT2)`

probably around line 85382

add this code under the line we found

```
Name (_PR0, Package(0x01)
                {
                    \_SB.PC00.I2C5.PTPL
                })
```

now, your file is seemd like this

```
                CreateDWordField (SBFB, 0x0C, SPED)
                CreateWordField (SBFG, 0x17, INT1)
                CreateDWordField (SBFI, 0x05, INT2)
                Name (_PR0, Package(0x01)
                {
                    \_SB.PC00.I2C5.PTPL
                })
                Method (_INI, 0, NotSerialized)  // _INI: Initialize
                {
                    TPGI = T1GI /* \T1GI */
                    If (CondRefOf (\_SB))
                    {
                        If ((OSYS < 0x07DC))
                        {
```

keep going down and find this code

```
Method (_CRS, 0, NotSerialized)  // _CRS: Current Resource Settings
                {
                    If ((OSYS < 0x07DC))
                    {
                        Return (SBFI) /* \_SB_.PC00.I2C2.TPL1.SBFI */
                    }

                    If ((TPLM == Zero))
                    {
                        Return (ConcatenateResTemplate (I2CM (I2CX, BADR, SPED), SBFG))
                    }

                    Return (ConcatenateResTemplate (I2CM (I2CX, BADR, SPED), SBFI))
                }
```

replace this code with 

```
Method (_CRS, 0, NotSerialized)  // _CRS: Current Resource Settings
                {
                   Return (ConcatenateResTemplate (I2CM (I2CX, BADR, SPED), SBFG))
                }
```
now,your file is seemd like this

```
Method (_STA, 0, NotSerialized)  // _STA: Status
                {
                    If ((TPLT == One))
                    {
                        Return (0x0F)
                    }

                    Return (Zero)
                }

                Method (_CRS, 0, NotSerialized)  // _CRS: Current Resource Settings
                {
                   Return (ConcatenateResTemplate (I2CM (I2CX, BADR, SPED), SBFG))
                }
            }

            Name (_DSD, Package (0x02)  // _DSD: Device-Specific Data
            {
                ToUUID ("f87a6d23-2884-4fe4-a55f-633d9e339ce1") /* Unknown UUID */, 
                Package (0x04)
                {
```

finally,turn to the header of this file

you can see many codes like `External (_SB_.XXXX.XXXX,XXXXXX)`

add this line of code anywhere

```External (\_SB.PC00.I2C5.PTPL, PowerResObj)```

after saving the file,run `iasl -ve -tc dsdt.dsl`

you can see some errors

after annotating these wrong codes,you will see the compiled dsdt

## fix broken keyboard interface

after fix the dsdt and boot,you will find microphone mute LED flickering 

you can add a udev rules to fix this problem

```sudo vim /etc/udev/rules.d/99-fixbrokenkeyboard.rules```

and write this

```
ACTION=="add|change", SUBSYSTEM=="input", ATTRS{id/vendor}=="2808", ATTRS{id/product}=="5662", ATTRS{name}=="*UNKNOWN*", ENV{LIBINPUT_IGNORE_DEVICE}="1"
```

in the last,run this command
```
sudo udevadm control --reload-rules
sudo udevadm trigger
```
