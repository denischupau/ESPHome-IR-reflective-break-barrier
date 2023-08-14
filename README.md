# Infrared Reflective Barrier based on ESPHome

## The problem

Some of the neighbours' dogs, when I'was not at home, were coming to relieve themselves on my lawn and driveway, it stinked, and I had my feet or wheels regularly full of it. But... nobody wanted to take responsibility.

So I had to take action, and confront them with proofs, and as I like to code and build eletronic things: let's DIY!

## The idea I came up with

A light barrier (with a laser, because, well, freakin' lasers!) across the entrance gate. The module would detect the breach of the beam and send an MQTT message to an existing server, and an automation would ring the doorbell and take a picture of the gate and its surroundings.

## The engineering

I would use an ESP-01 (ESP8266) module I had laying around, a laser or IR emitter with a corresponding receiver, both the same side of the gate, with a reflector the other side, to avoid wires running across the driveway.

After reading about freakin' lasers, I abandonned this idea: the laser light will certainly result difficult to tell appart from the daylight or beams of light from cars, and be (very?) difficult to point and align. Bye-bye cool laser stuff!

So let's go for an IR emitter (1 or more IR LEDs)! The good thing is that I had an old IR receptor from an old TV set, and to use it, I have to pulsate the IR light at 38kHz. No problem with PWM. And it looks like it can reach great distances up to 15m!

Reading about those receptors, they detect a 38kHz pulsed IR light, and when detected, they force a Low level on their output (open collector?). But after som time of receiving the 38kHz IR pulses, it ignores the signal and the output returns High (as if no signal was received).

That means the 38kHz carrier can't run continuously: it has to "blink" at 20-300Hz (modulation signal).

So the ESP01 would have to generate the 38kHz signal, and "blink" it at a rate of 20-300Hz and detect those "blinks" with an interrupt on a pin and fire an event when it misses more than 2 blinks for some 10ms (beam broken).

The 10ms timing looks OK: it is low enough to detect a fast moving object/animal/person, and high enough to be able to include at least 3 "blinks" at a low  blinking speed.


## POC, problems and solutions

The quick POC on a ATmega32u4 (Arduino Pro Micro), shows that everything works very well indeed!
* the Ardnuino function "sound(38000)" generates the 38kHz precisely (hardware PWM) and it's easy to stop and run it in the "loop()" function.
* the detection of the "blinks" works very well, each blink fires one interrupt, and if we are missing more than 2 interrupts in 10ms, it lights a LED up,
* the distance to the reflector is 4m, which is enough for the gate.

Problems:
* as those IR sensors are REALLY sensitive to the 38kHz IR signal, it was picking up the carrier directly from the IR LEDs, not from the reflection, through the grey PVC tube. So I had to isolate the emitter really well in a piece of black tubing (many PVC tubes are permeables to IR light it seems...),
* it seems that not all the IR LEDs work equally with those sensors, and the sensitivity ( and so, distance to the reflector) varies greatly with the wavelength of the IR light... so careful which LEDs you buy, it must match the wavelength of the sensor! (which can be difficult to find when salvaged from an old TV set)
* the ESP8266 does not have hardware PWM: it's software generated! And when implementing it on this module, the 38kHz was ... not 38kHz at all and varying a lot with the load on the mono-core micro (Wifi seems to generate a lot of interrupts and the other tasks as well).
* and with the Arduino SDK (MQTT IoT), I couldn't find a way to execute the "loop()" or "Ticker" functions at a 2-20ms period: Arduino seems too slow.
* to be more efficient I tried to program the ESP01 writting a FreeRTOS MQTT solution to see if it would be better: but the Espressif SDK is so old and unmaintained that nothing compiles (Python 2.4, GCC 11, etc.). It was too time consuming for me to debug.

Solutions:
* after searching for a solution to accurately generate the 38kHz pulses, I came across this in the ESP8266 specs: it has DMA and the I2S can be driven through DMA, and luckily the I2S data pin (SD) is accessible directly on the ESP01!
* somebody already modified the I2S DMA code to make a programmable Clock Generator running a constant pattern: cool! 
* the ESPhome framework allows to run custom code, and it has everything to make an IoT, up to the Over-the-Air update of the firmware, neat!
* somebody had already made a similar barrier with another microcontroller, so I could derive the electronics schematics for the "analog" part.

I was good to go!

## Implementation

So I'm creating a Wifi IoT that creates a barrier using a blinking IR beam coupled with an IR remote sensor both on the same side of the barrier, a reflector is on the other side of the barrier in front of them,
so there is no need to have a cable running between the 2 sides of the barrier.

* The IoT barrier (beam-breaking IoT) is running ESPHome on an ESP-01 1MB (ESP8266) with some custom code.
* It generates a 38kHz carrier for an IR emitter (IR LEDs driven by a general purpose, salvaged, NPN transistor) via the GPIO3 (RX and I2S SD) pin.
* The 38kHz square wave is accurately generated by outputing a 0xAAAAAAAAAAAAAAAA pattern via the I2S peripheral (DMA driven).
* This 38kHz carrier is "blinking" at a rate of 30-70Hz to avoid the IR sensor (here HS0038 or other from an old TV set) ignoring the 38kHz carrier after a while: the carrier can NOT be continuously running with this sensor.
* The builtin blue LED (LED_BUILTIN) of the ESP-01 goes ON when the barrier is broken, and is OFF when the barrier is working fine without obstacle. That facilitates the aiming and placement!
* This setup works with a distance between 30cm - 4m from the IR emitter/receiver to the reflector.

I built the circuit from the schematics in the included image, with the code included here and the YAML file to build the ESPhome firmware.

## Some problems I had

* close to the IR emitter (30-50cm), more or less all materials are reflective to IR and the barrier is never broken.
* every now and then the ESPhome IoT goes "unavailable", that doesn't seem to influence the fiability of this IoT though.
* to update OTA, it works after several attempts.

## Some details

The ESP8266 pins for I2S are
* Data bits (SD) = GPIO3/RX0     (directly accessible on the ESP-01)
* Data bit-clock (SCK) = GPIO15  (NOT directly accessible on the ESP-01's)
* Word select (WS) = GPIO2/TX1   (directly accessible, though not used here)

The connections
* GPIO3 (RX0) is connected to the base of the transistor via a resistor on the emitter board,
* GPIO0 is connected to the IR recptor via a resistor on the receptor board.
* GPIO0 and GPIO3 are connected to their respective bord (emitter board & receptor board) through a 4-wire cable (GND, 5v, GPIO0, GPIO3) where the emitter/receptor are outside the house,
* 5V from an old USB phone charger is applied on the VCC connector,
* the other connector (programming serial port) can be used with GPIO0 connected to ground (flashing mode) before resetting, then flash the firmware.

In the source code:
* The blinking that works well in my tests is BLINK_OFF at 2xBLINK_ON, with BLINK_ON_MILLIS around 6 to 20ms.
* For some other values of "blinking" it works well for a while (15-30min) then the sensor begins to ignore the IR carrier signals, and the beam appears as always broken.

The hardware:
* All the components were salvaged from old electronics, but the ESP01 and its 3.3v regulator which are new, old stock of mine.
* The whole thing is powered on 5V from an old phone USB phone charger (have quantites of them!).

## Other solutions and future improvements

Other solutions I considered, and may consider in the future:
* use a small microcontroller with hoardware PWM (i.e. ATTiny85) connected to the ESP01, to generate the 38kHz carrier with the arduino function "tone(38000);" and its modulation, and also make the detection of the breach and generate an event (pin transition to simulate a button press/release, serial message) so ESPhome only has to report on this event when it can, no time-sensitive task.
* as in a few of the resources: 2 NE555 to generate the 38kHz (38,461Hz) carrier and its modulation, and also the detection, old school, but efficient and reliable!

Imrovements:
* make the I2S DMA pattern generate the "blinks" also by inserting 0s in the 38kHz pattern

## Resources and inspiration

Thanks to them all for their good work:
* I2S inspiration code: https://shepherdingelectrons.blogspot.com/2018/10/the-ir-egg-voice-controlled-esp8266-i2s.html
* I2S Clock Pattern Generator code: https://github.com/roberttidey/espI2sClockGen
* I2S Clock Pattern Generator website: https://www.instructables.com/Esp8266-Clock-and-Pulse-Generator/
* https://www.hackster.io/mova2/long-range-beam-break-sensor-with-reflector-panel-4dfc48
* A light barrier, with 3 NE555, with lots of details: https://www.electroschematics.com/infrared-light-barrier/
* Another light barrier with 4 NE555: https://espenandersen.no/light-barrier/


## How to program it

Put the 3 files (irbarrier.h, and the 2 i2sTXcircular.* in an "irbarrier01" folder) in your ESPHome config folder (usually where your YAML file is).

In the YAML file for this ESPHome ESP01 1MB device, add:
```YAML
esphome:
  name: irbarrier01
  friendly_name: IRbarrier01
  includes:
  - irbarrier01/

binary_sensor:
  - platform: custom
    lambda: |-
      auto irbarrier = new MyIRBarrier();
      App.register_component(irbarrier);
      return {irbarrier};

    binary_sensors:
      name: "IR Barrier"
```

