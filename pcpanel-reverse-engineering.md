# PCPanel Reverse Engineering

2021-02-13: I have ordered a unit but it hasn't been shipped yet so all I can do is look at the driver.

2021-02-16: I figured out what the VolumeTracking does.

2021-04-22: Got output of lsusb from a user that already got his PCPanel Pro.

## Looking at the driver

I started by looking at the driver installer using binwalk and strings. I tried to extract stuff from it but I couldn't find anything useful.

I extracted files from the installer using `innoextract`. Most interesting to me was `PCPanel.exe` of course but I also noticed a folder called `jre/` indicating that this application was written in java.

I used binwalk again but this time on the installed `PCPanel.exe` and it found a whole bunch of files. I extracted it and started looking through the files. Apparently an exe file can simply be unzipped.

In an assets folder I found some images the application uses.

I was most drawn to a folder called `hid/` indicating that this was a hid device, I was a little worried it was just a serial device like I think the first wooden PCPanel was. Inside the hid folder I found a bunch of `.class` files which fits in line with a java application.

Using the java decompiler `cfr` on some of these files I could figure out the USB protocol. Most important ones are `DeviceCommunicationHandler.class` which handles incoming data from the PCPanel (knob turning and button clicking) and `OutputInterpreter.class` which handles sending data to the PCPanel (setting RGB state).

## Findings

Incoming data from PCPanel is 3 bytes long, the first being a message type (knob turn or button press), the second being the knob in question (I'm guessing 0-3, but I'll have to wait for my unit to test) and finally the last byte being the value of the knob turn or button press/release.

```
KNOB_CHANGE=1
BUTTON_CHANGE=2

--------------------------------------------------
| KNOB_CHANGE or BUTTON_CHANGE | KNOB_ID | VALUE |
--------------------------------------------------
```

The outgoing RGB data is a little bit more complicated. The following constants were at the top of the decompiled function:

```
private static final byte[] OUTPUT_CODE_INIT = new byte[]{1};
private static final byte OUTPUT_CODE_RGB = 2;
private static final byte OUTPUT_CODE_LED_MAX_CURRENT = 3;
private static final byte OUTPUT_CODE_RGB_ALL_KNOBS = 0;
private static final byte OUTPUT_CODE_RGB_SINGLE_KNOB = 1;
private static final byte OUTPUT_CODE_RGB_FULL_DATA = 0;
private static final byte OUTPUT_CODE_RGB_RGB = 1;
private static final byte OUTPUT_CODE_RGB_HSV = 2;
private static final byte OUTPUT_CODE_RGB_RAINBOW = 3;
private static final byte OUTPUT_CODE_RGB_WAVE = 4;
private static final byte OUTPUT_CODE_RGB_BREATH = 5;
```

There is a function called `sendInit(String deviceSerialNumber)` which just sends that one byte `OUTPUT_CODE_INIT`. I'm not sure if this get's called once per device or everytime you are gonna send something, I'll have to look at the traffic in wireshark once I get my unit.

All other outgoing data starts with 2 as message type (`OUTPUT_CODE_RGB`), followed by the type of RGB data is coming. This can be RGB or HSV static data as well as some animations being RAINBOW, WAVE and BREATH. What follows after that depends on the type.

Animations:

```
RAINBOW:
-------------------------------------------------------------------
| 2 | 3 | PHASE_SHIFT | SATURATION | BRIGHTNESS | SPEED | REVERSE |
-------------------------------------------------------------------

WAVE:
--------------------------------------------------------------------
| 2 | 4 | HUE | SATURATION | BRIGHTNESS | SPEED | REVERSE | BOUNCE |
--------------------------------------------------------------------

BREATH:
-------------------------------------------------
| 2 | 5 | HUE | SATURATION | BRIGHTNESS | SPEED |
-------------------------------------------------
```

Static:

```
SendRGB:
-----------------------------------
| 2 | 1 | 1 | KNOB_ID | R | G | B |
-----------------------------------

SendRGBAll:
-------------------------
| 2 | 1 | 0 | R | G | B |
-------------------------

SendHSV:
-----------------------------------
| 2 | 2 | 1 | KNOB_ID | H | S | V |
-----------------------------------

SendHSVAll:
-------------------------
| 2 | 2 | 0 | H | S | V |
-------------------------
```

I can also see functions `sendLEDMaxCurrent` and `sendFullLEDData` that I will need to investigate further by sniffing the USB traffic.

But from what I can tell `sendFullLEDData` works as following:
```
        |-- Repeats N times over number of knobs --|-- Repeats N times over number of "VolumeTrack" --|
-------------------------------------------------------------------------------------------------------
| 2 | 0 |      1     |    R    |    G    |    B    |                       1/0                        |
-------------------------------------------------------------------------------------------------------
```

~~What I'm unsure of is what the meaning of volume track is in this case, maybe it's only relevant for the PCPanel Pro which has sliders as well or maybe it tells the knobs to only light part of the ring that follows the potentiometer, I don't know if the hardware is capable of it.~~

VolumeTrack simply sets the knobs to set the led brightness depending on how high the volume knob is turned.

## USB info

PCPanel pro:
```
Bus 001 Device 006: ID 0483:a3c5 STMicroelectronics PCPanel Pro 1.0
Couldn't open device, some information will be missing
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               2.00
  bDeviceClass            0
  bDeviceSubClass         0
  bDeviceProtocol         0
  bMaxPacketSize0        64
  idVendor           0x0483 STMicroelectronics
  idProduct          0xa3c5
  bcdDevice            2.00
  iManufacturer           1
  iProduct                2
  iSerial                 3
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength       0x0029
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0
    bmAttributes         0xc0
      Self Powered
    MaxPower              100mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           2
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      0
      bInterfaceProtocol      0
      iInterface              0
        HID Device Descriptor:
          bLength                 9
          bDescriptorType        33
          bcdHID               1.11
          bCountryCode            0 Not supported
          bNumDescriptors         1
          bDescriptorType        34 Report
          wDescriptorLength      33
         Report Descriptors:
           ** UNAVAILABLE **
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0040  1x 64 bytes
        bInterval               1
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x01  EP 1 OUT
        bmAttributes            3
          Transfer Type            Interrupt
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0040  1x 64 bytes
        bInterval               1
```
