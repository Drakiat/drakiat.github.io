---
title: "Turning a motorized mount smart with a Raspberry Pi Pico and CC1101 module"
author: "Felipe Tapia Sasot"
date: 2025-12-19
categories: Cybersecurity
tags: Dooya,CC1101,Arduina,RF
---
I recently purchased a motorized mount for my TV to sit above the fireplace (I know, r/TVTooHigh ...). This motorized mount comes with a remote with an UP, DOWN and STOP commands. As part of my home automation setup, I use a Google Home to connect to most of my device, hence I had to find a way to integrate this mount into my setup.

By opening the remote battery casing and looking at the FCC-ID, I found out that the signal transmits on the 433.92mHz frequency. I used my Flipper Zero to capture and examine the signal, which decoded it with no issues. It turns out the signal is using the Dooya protocol, which is used for blind shutters, and other home appliances.

```
Filetype: Flipper SubGhz Key File
Version: 1
Frequency: 433920000
Preset: FuriHalSubGhzPresetOok650Async
Protocol: Dooya
Bit: 40
Key: 00 00 00 53 82 00 01 33

```
In terms of tools, I went with the Raspberry Pi Pico 2W microcontroller, and a CC1101 433mhz module. I also found [this](https://www.printables.com/model/537529-raspberry-pi-pico-rp2040-cc1101-tool-chassis/related) cool case which I 3D printed for my devices.

<p><img src="/images/case.png" alt="3D-printed case" /></p>
<br>

## Wiring

I used the following wiring in my setup.

```
| CC1101 | Raspberry Pi Pico |
| ------ | ----------------- |
| GND    | GND               |
| VCC    | 3v3 OUT           |
| GDO0   | GP20              |
| CSN    | GP17              |
| SCK    | GP18              |
| MOSI   | GP19              |
| MISO   | GP16              |
| GPO2   | Unused            |
```
<p><img src="/images/wiring.png" alt="Wiring photo" /></p>
<br>
Please note that your CC1101 may have different pinout than mine.

## Signal Capture and Replay

After much trial and error with CircuitPython and MicroPython to replay the signal, I was not able to get the ASK/OOK signals from the Dooya remote to trigger the device, I mostly used resources from [this repo](https://github.com/unixb0y/CPY-CC1101) which let met transmit strings, but I did not find a way to replay my remote's signal.

[This article](https://www.elektroda.com/news/news4157129.html) seemed promising, in which they used an ESP32 with ESPHome in order to capture and replay the Dooya signals. However, I did not want to install Home Assistant on my setup. It was nonetheless interesting to see the libraries they were using, in particular **RadioLib**.

RadioLib is a powerful library meant for ESP32 and Arduinos, but as it turns out is also compatible with Raspberry Pi Picos!

[This discussion on the RadioLib repo](https://github.com/jgromes/RadioLib/discussions/842) was very helpful in getting the initial setup with it. I was able to use these snippet of code to transmit signals, and then see them on my Flipper. 

## Implementation

Using RadioLib, I decided to make my project as follows: 

-A function that captures raw signals for 5 seconds
```cpp
void captureRaw(uint32_t captureMs) {
int16_t st = radio.receiveDirectAsync();
if(st != RADIOLIB_ERR_NONE) {
Serial.print(F("receiveDirectAsync() failed, code "));
Serial.println(st);
return;
}
capCount = 0;
lastEdgeUs = micros();
lastLevelHigh = digitalRead(GDO0_PIN);
capturing = true;
attachInterrupt(digitalPinToInterrupt(GDO0_PIN), onGdo0Change, CHANGE);
uint32_t t0 = millis();
while(capturing && (millis() - t0 < captureMs)) {
yield();
}
capturing = false;
detachInterrupt(digitalPinToInterrupt(GDO0_PIN));
radio.standby();
// Append a tail duration for the last level (nice for replays)
uint32_t now = micros();
uint32_t dt = now - lastEdgeUs;
if(capCount < MAX_EDGES) {
capTimings[capCount++] = lastLevelHigh ? (int32_t)dt : -(int32_t)dt;
}
}
```
- Press the button on my remote
- Print via serial the array of edges to copy and transmit
```cpp
void printCapturedAsArray() {
noInterrupts();
uint16_t n = capCount;
interrupts();
Serial.println();
Serial.print(F("Captured edges: "));
Serial.println(n);
Serial.println(F("const int32_t RAW[] = {"));
for(uint16_t i = 0; i < n; i++) {
int32_t v;
noInterrupts();
v = capTimings[i];
interrupts();
Serial.print(F(" "));
Serial.print(v);
if(i + 1 < n) Serial.print(',');
if((i % 8) == 7) Serial.println();
}
Serial.println();
Serial.println(F("};"));
Serial.println(F("const size_t RAW_LEN = sizeof(RAW)/sizeof(RAW[0]);"));
Serial.println();
}
```

- We can then paste the array into a variable, and attempt to send them via a transmit function

```cpp
const int32_t RAW[] = {

687434, -1513, 347, -751, 708, -359, 342, -771,

696, .........

};

const size_t RAW_LEN = sizeof(RAW)/sizeof(RAW[0]);
```

```cpp
void transmitRawTimings(const int32_t* raw, size_t rawLen, uint8_t repeats, uint32_t gapUs) {
if(raw == nullptr || rawLen == 0) {
Serial.println(F("transmitRawTimings: empty RAW"));
return;
}
for(uint8_t r = 0; r < repeats; r++) {
int16_t st = radio.transmitDirectAsync();
if(st != RADIOLIB_ERR_NONE) {
Serial.print(F("transmitDirectAsync() failed, code "));
Serial.println(st);
return;
}
// Drive the line connected to CC1101 GDO0
pinMode(GDO0_PIN, OUTPUT);
for(size_t i = 0; i < rawLen; i++) {
int32_t v = raw[i];
bool level = (v > 0);
uint32_t us = (v > 0) ? (uint32_t)v : (uint32_t)(-v);
digitalWrite(GDO0_PIN, level ? HIGH : LOW);
// Split long delays into safe chunks
while(us > 0) {
uint32_t chunk = (us > 16000) ? 16000 : us;
delayMicroseconds(chunk);
us -= chunk;
yield();
}
}
// Idle low (common for OOK)
digitalWrite(GDO0_PIN, LOW);
radio.standby();
pinMode(GDO0_PIN, INPUT);
// Inter-frame gap
uint32_t g = gapUs;
while(g > 0) {
uint32_t chunk = (g > 16000) ? 16000 : g;
delayMicroseconds(chunk);
g -= chunk;
yield();
}
}
}
```
Using these functions, I was able to get my mount to move up and down programmatically. Sweet!

## Integration with Home Automation

Next I needed to connect the microcontroller to my Wi-Fi Network, and link it to my Google Home. I used two free online services for that: IFTTT and Adafruit IO.

## Adafruit IO Setup

After creating a free account on Adafruit, you can need to create a Feed and a Dashboard. In my dashboard I made a simple Toggle Switch with UP/DOWN values.

<p><img src="/images/adafruit.png" alt="Adafruit dashboard" /></p>
<br>

## IFTTT Setup

On IFTTT, you need to link both your Google Assistant and your Adafruit Account. You can then set up a new Applet where 
IF = Google Assistant -> "Activate Scene" 
THEN = Adafruit -> "Send Data to a Feed"
<p><img src="/images/ifttt.png" alt="IFTTT setup" /></p>
<br>

## Connecting the Pico

Once it is all setup, you need to connect you Pico to Wi-Fi and Adafruit. In order to talk to Adafruit using the MQTT protocol, I used built-in libraries, as well as PubSubClient.h by Nick O'Leary, which was easily installable through the Arduino "Manage Libraries" interface.

## Code

The full .ino code is published [here](https://github.com/Drakiat/c1101-dooya/tree/main) and should be easily modifiable for other use cases.

<p><img src="/images/signal.gif" alt="Signal replay GIF" /></p>
<br>

