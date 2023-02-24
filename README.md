---
https://squidgeefish.com/projects/a1175-battery-hacking/
---

## Background
With the release of Intel-based MacBooks in 2006, Apple started using "smart battery monitor" chips that track current in and out of the battery to provide more realistic battery life estimations. However, this means that, unlike the PowerPC laptops, failed battery cells cannot simply be replaced: disconnecting cells triggers a permanent failure flag in the battery monitor, disabling the battery. I hate it when that happens...

The battery monitor chip in at least the 2006-era batteries is a TI [bq20z80](https://www.ti.com/lit/ds/symlink/bq20z80.pdf), operating in concert with a TI [bq29312](https://www.ti.com/lit/ds/symlink/bq29312.pdf) battery management IC. The bq20z80 provides an SBS (Smart Battery System) interface over SMBUS (basically I2C), and the bq29312 handles the actual monitoring of the battery (charge/discharge monitoring, per-cell voltage monitoring, etc.).  
After perusing Charlie Miller's seminal Battery Firmware Hacking presentation ([DEFCON slides](https://defcon.org/images/defcon-19/dc-19-presentations/Miller/DEFCON-19-Miller-Battery-Firmware-Hacking.pdf), [PDF writeup](https://papers.put.as/papers/macosx/2011/Battery-Firmware-Hacking.pdf)), I learned that the bq2080 has three modes of operation ("sealed," "unsealed," and "full access"), the latter two of which are guarded by 32-bit passwords which Apple has left at their factory default values. I figured that this meant I should be able to reset the permanent failure flag on a dead battery with an Arduino, but I didn't have an immediate excuse to try it out...

Fast-forward a few years, and the A1175 battery in my 15" MacBook Pro refused to power up after being left sitting on a shelf for a year. I wasn't using this laptop too much at that point anyway, but since Apple downclocks MacBooks pretty heavily when no (working) battery is installed, it is much more convenient to have a working battery.  
I had some spare LiPo cells out of a 13" MacBook Air (only one out of the four cells had failed - the others were fine at around 4V each) and wired them up to the controller board from my dead battery. Yes, this is less capacity than the original battery, but I mainly wanted to provide enough power to move the laptop between chargers and keep OS X from downclocking the laptop.

![Re-celled A1175 battery with spacer](/assets/{{page.slug}}/battery-spacer.jpg)

The piece of plastic flooring in that picture keeps the controller board from being able to slide back. Originally, the battery cells filled the whole space behind the controller board in a 3S2P configuration, but I only had space for 3S1P with the MacBook Air cells.

I then added clearing the permanent failure flag on that controller to my list of things to do in my copious amounts of spare time.

Fast-forward a few more years, and I actually got around to it!

## Wiring

![A1175 pinout](/assets/{{page.slug}}/pinout.jpg)

To connect to the battery, I tried sticking normal Dupont-style jumper wires into the slots, but I had a terrible time getting the springs on the connector to hold onto them (some people have had more luck with diode legs or solid-core breadboarding jumpers). After getting an I2C response from address `0xB` once, I just pulled the battery connector assembly out of a dead 15" MacBook Pro and soldered jumpers to `SMBUS_BATT_SCL` and `SMBUS_BATT_SDA`, which (along with `GND`) are the only connections necessary for this project.

![Arduino wired to A1175 controller board](/assets/{{page.slug}}/battery-programming.jpg)

## SBS Poking
I made some convenience functions to perform standard SMBUS transactions with 16-bit words and 4-byte data blocks (full code listing in the [Code](#code) section).
```c
uint16_t readWord(uint8_t address);
uint32_t readLong(uint8_t address);
void writeWord(uint8_t address, uint16_t word);
```
TI document [SLUU276](https://www.ti.com/lit/er/sluu276/sluu276.pdf) details all the SBS commands that the bq20z80 supports.

### Temperature and Voltage
I started out with a pair of commands that are accessible in any battery mode.  
Command `0x08` gives temperature in deciKelvins (from that thermocouple), and command `0x09` gives the battery voltage in millivolts.
```c
void loop() {
    Serial.print(readWord(0x09)*0.001);
    Serial.print("V  ");
    Serial.print(readWord(0x08)*0.1 - 273.15);
    Serial.println("°C");
    delay(1000);
}
```
```
12.42V  22.05°C
12.42V  22.05°C
12.42V  21.95°C
12.42V  21.95°C
65.54V  6280.35°C
65.54V  6280.35°C
65.54V  6280.35°C
```
Note to self - don't unplug the battery from the Arduino while talking to it...

### Unsealed Mode
> Unsealing is a two-step command performed by writing the first word of the *UnSealKey* to *ManufacturerAccess* followed by the second word of the *UnSealKey* to *ManufacturerAccess*. 

*ManufacturerAccess* is a standard SBS command `0x00`, and we know that the password is `0x36710414`, with the least significant word sent first.  
To know if we've successfully escaped the sealed mode, we need to see if the `SS` bit (bit 13) of the *OperationStatus* command `0x54` is cleared.

```c
writeWord(0x00, 0x0414);
writeWord(0x00, 0x3672);

Serial.print("OperationStatus: ");
Serial.println(readWord(0x54), HEX);
```
```
4010
```
Bit 13 (`SS`) is clear, but bit 14 (`FAS` - Full Access) is still set, meaning that we aren't in the Full Access mode yet (logical, as we have not sent the command to enter that mode).

Note that *OperationStatus* and all other extended SBS commands return garbage values in sealed mode. I didn't know this at first, so I spent quite a while trying to figure out why *PFStatus* (another extended command - `0x53`) was giving me nonsensical errors...

### Permanent Fail Clear
> **Permanent Fail Clear: (1) 0x2673 then (2) 0x1712**, Instructs the bq20z80 to clear the Permanent Failure Status, clear the Permanent Failure Flag, clear the `SAFE` and `SAFE#` pins, and unlock the data flash for writes.  
> This function is only available when the bq20z80 is *Unsealed*

```c
writeWord(0x00, 0x2673);
writeWord(0x00, 0x1712);

Serial.print("PFStatus: ");
Serial.println(readWord(0x53), HEX);
```
```
0
```
Alright - all flags are clear!

I resealed the battery by writing `0x0020` to *ManufacturerAccess*.

### Bonus - AccessKeys via Full Access mode
Block commands *UnsealKey* (`0x60`), *FullAccessKey* (0x61), and *PFKey* (0x62) are only accessible in the Full Access mode. In our case, we got lucky and Apple left everything as the default values. They return the 32-bit keys in little-endian format.

```c
Serial.print("PFKey: ");
Serial.println(readLong(PF_KEY), HEX);

Serial.print("UnSealKey: ");
Serial.println(readLong(UNSEAL_KEY), HEX);

Serial.print("FullAccessKey: ");
Serial.println(readLong(FULL_ACCESS_KEY), HEX);
```
```
PFKey: 17122673
UnSealKey: 36720414
FullAccessKey: FFFFFFFF
```

## Conclusion
Because I'd let the battery sit on a shelf for a few years after installing the MacBook Air cells, they were a bit low when I brought it out to start poking at the SBS interface (~2.5V each or so...). I trickle-charged them up to 3.3V each with a NiMH charger set to 0.1A and then charged them the rest of the way at 0.5A with a LiPo charger. This seemed to work, but the LED battery gauge on the back of the battery didn't light up despite the cells being at 4.2V each. This wound up resolving itself after I made a successful I2C connection to the battery: all the lights lit up, and the button has worked since.

In order to re-calibrate the battery, I followed the Apple procedure at the bottom of [this iFixit page](https://help.ifixit.com/article/265-battery-calibration). To discharge the battery, I spent about an hour digging through old files and discovering that, in fact, nostalgia is not what it used to be.

![MacBook Pro with battery installed showing System Profiler screen](/assets/{{page.slug}}/happy-mac.jpg)

System Profiler now shows the battery as having about 1400mAh of capacity or so and wants me to "Service Battery," but it seems good enough to me...

## Code
```c
#include "Arduino.h"
#include "Wire.h"

#define I2C_ADDR 0xB

#define MANUFACTURER_ACCESS 0x00
#define TEMPERATURE 0x8
#define MILLIVOLTS 0x9
#define PF_STATUS 0x53
#define OPERATION_STATUS 0x54
#define UNSEAL_KEY 0x60
#define FULL_ACCESS_KEY 0x61
#define PF_KEY 0x62

uint16_t readWord(uint8_t address);
uint32_t readLong(uint8_t address);
void writeWord(uint8_t address, uint16_t word);

void setup()
{
    Serial.begin(115200);
    while (!Serial);
    Wire.begin();

    // Unseal battery - 36720414
    writeWord(MANUFACTURER_ACCESS, 0x0414);
    writeWord(MANUFACTURER_ACCESS, 0x3672); 

    Serial.print("OperationStatus: ");
    Serial.println(readWord(OPERATION_STATUS), HEX);

    Serial.print("PF Status: ");
    Serial.println(readWord(PF_STATUS), HEX);

    // Full access mode
    writeWord(MANUFACTURER_ACCESS, 0xFFFF);
    writeWord(MANUFACTURER_ACCESS, 0xFFFF);
    
    Serial.print("OperationStatus: ");
    Serial.println(readWord(OPERATION_STATUS), HEX);

    Serial.print("PFKey: ");
    Serial.println(readLong(PF_KEY), HEX);

    Serial.print("UnSealKey: ");
    Serial.println(readLong(UNSEAL_KEY), HEX);

    Serial.print("FullAccessKey: ");
    Serial.println(readLong(FULL_ACCESS_KEY), HEX);

    // 0x17122673 is the Permanent Failure Clear key
    writeWord(MANUFACTURER_ACCESS, 0x2673);
    writeWord(MANUFACTURER_ACCESS, 0x1712);

    Serial.print("PF Status: ");
    Serial.println(readWord(PF_STATUS), HEX);

    // Seal battery
    writeWord(MANUFACTURER_ACCESS, 0x0020);

    Serial.println("Voltage Temperature");
}

void loop()
{
    Serial.print(readWord(MILLIVOLTS)*0.001);
    Serial.print("V  ");
    Serial.print(readWord(TEMPERATURE)*0.1 - 273.15);
    Serial.println("°C");
    delay(1000);
}

void writeWord(uint8_t address, uint16_t word)
{
    Wire.beginTransmission(I2C_ADDR);
    Wire.write(address);
    Wire.write(word & 0xFF);
    Wire.write((word >> 8) & 0xFF);
    Wire.endTransmission();
}

uint16_t readWord(uint8_t address)
{
    Wire.beginTransmission(I2C_ADDR);
    Wire.write(address);
    Wire.endTransmission(false);

    Wire.requestFrom(I2C_ADDR, 2, true);
    uint16_t retVal = Wire.read();
    retVal |= Wire.read() << 8;
    Wire.endTransmission();

    return retVal;
}

uint32_t readLong(uint8_t address)
{
    Wire.beginTransmission(I2C_ADDR);
    Wire.write(address);
    Wire.endTransmission(false);

    Wire.requestFrom(I2C_ADDR, 5, true);
    Wire.read(); // Ignoring the length byte (this is a block-read transaction)
    uint32_t retVal = Wire.read();
    retVal |= ((uint32_t) Wire.read() << 8);
    retVal |= ((uint32_t) Wire.read() << 16);
    retVal |= ((uint32_t) Wire.read() << 24);
    Wire.endTransmission();

    return retVal;
}
```
```
OperationStatus: C043
PF Status: 0
OperationStatus: 8043
PFKey: 17122673
UnSealKey: 36720414
FullAccessKey: FFFFFFFF
PF Status: 0
Voltage Temperature
12.42V  19.55°C
12.42V  19.55°C
12.42V  19.55°C
12.42V  19.55°C
12.42V  19.55°C
...
```
