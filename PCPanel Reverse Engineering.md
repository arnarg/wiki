# PCPanel Reverse Engineering

2021-02-13: I have ordered a unit but it hasn't been shipped yet so all I can do is look at the driver.

## Looking at the driver

I started by looking at the driver installer using binwalk and strings. I tried to extract stuff from it but I couldn't find anything useful.

Then I ran the installer using wine and after that was finished I could find files in `$WINEPREFIX/drive_c/users/arnar/Local\ Settings/Application\ Data/PCPanel\ Software/`. Most interesting to me was `PCPanel.exe` of course but I also noticed a folder called `jre/` indicating that this application was written in java.

I used binwalk again but this time on the installed `PCPanel.exe` and it found a whole bunch of files. I extracted it and started looking through the files.

In an assets folder I found some images the application uses.

I was most drawn to a folder called `hid/` indicating that this was a hid device, I was a little worried it was just a serial device like I think the first wooden PCPanel was. Inside the hid folder I found a bunch of `.class` files which fits in line with a java application.

Using the java decompiler `cfr` on some of these files I could figure out the USB protocol. Most important ones are `DeviceCommunicationHandler.class` which handles incoming data from the PCPanel (knob turning and button clicking) and `OutputInterpreter.class` which handles sending data to the PCPanel (setting RGB state).

My findings:

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

RAINBOW:
```
-------------------------------------------------------------------
| 2 | 3 | PHASE_SHIFT | SATURATION | BRIGHTNESS | SPEED | REVERSE |
-------------------------------------------------------------------
```
WAVE:
```
--------------------------------------------------------------------
| 2 | 4 | HUE | SATURATION | BRIGHTNESS | SPEED | REVERSE | BOUNCE |
--------------------------------------------------------------------
```
BREATH:
```
-------------------------------------------------
| 2 | 5 | HUE | SATURATION | BRIGHTNESS | SPEED |
-------------------------------------------------
```

Static:

SendRGB:
```
-----------------------------------
| 2 | 1 | 1 | KNOB_ID | R | G | B |
-----------------------------------
```
SendRGBAll:
```
-------------------------
| 2 | 1 | 0 | R | G | B |
-------------------------
```
SendHSV:
```
-----------------------------------
| 2 | 2 | 1 | KNOB_ID | H | S | V |
-----------------------------------
```
SendHSVAll:
```
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

What I'm unsure of is what the meaning of volume track is in this case, maybe it's only relevant for the PCPanel Pro which has sliders as well or maybe it tells the knobs to only light part of the ring that follows the potentiometer, I don't know if the hardware is capable of it.
