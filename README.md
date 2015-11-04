## IO Plugins

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/rwaldron/io-plugins?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

(This document is a work in progress)

An IO Plugin is any class whose instances implement a [Firmata](https://github.com/jgautier/firmata) compatible interface. 
For an in-depth case study of creating an IO plugin, [read about the design and creation of the Galileo-IO Plugin here](http://bocoup.com/weblog/intel-galileo-javascript-nodejs/).

IO Plugins are used extensively by [Johnny-Five](https://github.com/rwaldron/johnny-five) to communicate with non-Arduino based hardware but may also be used independently if desired.

### Available IO Plugins

The following platform IO Plugins are currently available:

<!--extract-start:ioplugins-->
- Arduino StandardFirmata
  - [Firmata.js](https://github.com/jgautier/firmata)
- BeagleBone Black
  - [BeagleBone-IO](https://github.com/julianduque/beaglebone-io)
- Blend Micro
  - [Blend-Micro-IO](https://github.com/noopkat/blend-micro-io)
- BoardIO (Generic IO Plugin class to make your own!)
  - [Board-IO](https://github.com/achingbrain/board-io)
- Electric Imp
  - [Imp-IO](https://github.com/rwaldron/imp-io/)
- Intel Galileo, Edison
  - [Galileo-IO](https://github.com/rwaldron/galileo-io/)
  - [Edison-IO](https://github.com/rwaldron/edison-io/) (This is an alias module to Galileo-IO)
- LightBlue Bean
  - [Bean-IO](https://github.com/monteslu/bean-io/)
- Linino One
  - [Nino-IO](https://github.com/rwaldron/nino-io/)
- Particle Core & Particle Photon
  - [Particle-IO](https://github.com/rwaldron/particle-io/)
- pcDuino
  - [pcDuino-IO](https://github.com/rwaldron/pcDuino-io/)
- Pinoccio
  - [Pinoccio-IO](https://github.com/soldair/pinoccio-io/)
- Raspberry Pi
  - [Raspi-IO](https://github.com/nebrius/raspi-io)
- RemoteIO (Wrapper to remote control another IO class)
  - [Remote-IO](https://github.com/monteslu/remote-io)
- Tessel 2
  - [Tessel-IO](https://github.com/rwaldron/tessel-io)


<!--extract-end:ioplugins-->

### Minimum Plugin Class Requirements

The plugin must...

- Be a constructor function that defines a prototype object
- Be a subclass of [EventEmitter](http://nodejs.org/api/events.html#events_class_events_eventemitter)
- Initialize instances that must...
    - asynchronously emit a "connect" event when the connection to a physical device has been made.
    - asynchronously emit a "ready" event when the handshake to the physical device is complete.
    - include a property named `isReady` whose initial value is `false`. `isReady` must be set to `true` in the same or previous execution turn as the the "ready" event is emitted.
        - The process of establishing a connection and becoming "ready" is irrelevant to this document's purposes.
    - include a readonly property named `MODES` whose value is a frozen object containing the following property/values: `{ INPUT: 0, OUTPUT: 1, ANALOG: 2, PWM: 3, SERVO: 4 }` 
    - include a readonly property named `pins` whose value is an array of pin configuration objects. The indices of the `pins` array must correspond to the pin address integer value, eg. on an Arduino UNO digital pin 0 is at index 0 and analog pin 0 is index 14. See [mock-pins.js](https://github.com/rwaldron/johnny-five/blob/master/test/mock-pins.js) for a complete example.
        - each pin configuration object must contain the following properties and values: 
            - `supportedModes`: an array of modes supported by this pin, eg. 
                - `[0, 1, 2]` represents `INPUT`, `OUTPUT`, `ANALOG`. (Common analog read capable pins)
                - `[0, 1, 4]` represents `INPUT`, `OUTPUT`, `SERVO`.  (Common digital I/O capable pins) 
                - `[0, 1, 3, 4]` represents `INPUT`, `OUTPUT`, `PWM`, `SERVO`.  (Common digital I/O and PWM capable pins)
            - `mode`: the current mode this pin is set to.
            - `value`: the current value of this pin 
                - INPUT mode: property updated via the read loop
                - OUTPUT mode: property updated via *Write methods
            - `report`: 1 if reporting, 0 if not reporting
            - `analogChannel`: corresponding analogPin index (127 if none), eg. `analogChannel: 0` is `A0` whose index is `14` in the `pins` array.
    - include a readonly property named `analogPins` whose value is an array of pin indices that correspond to the analog pin indices in the `pins` array. 
- If an essential IO feature is not implemented or _cannot_ be implemented, the method _must_ throw. For example, the Raspberry Pi does not support analog inputs, if user code calls through to an `analogRead`, the program must throw as an irrefutable means of indicating non-support.
- If a non-essential IO feature is not implemented or _cannot_ be implemented, the method _must_ accept the expected arguments and indicate successful completion. For example, if it receives a callback, that callback _must_ be called asynchronously.

### Minimum API Requirements

**pinMode(pin, mode)**
- Set the mode of a specified pin to one of: 
```
INPUT: 0
OUTPUT: 1
ANALOG: 2
PWM: 3
SERVO: 4
```
- Pins with "A" prefix that are set to `INPUT` should actually store `ANALOG` (2) on the pin's mode property 
- In firmata.js and Firmata protocol, `ANALOG` mode is used for reading (input) on analog pins because only a pin address integer is sent over the wire, which means that `A0` is sent as `0`, `A1` as `1` and so on. This creates an ambiguity: which `0` and `1` are we sending, digital or analog? Then, for reporting, analog reads are separated: https://github.com/firmata/arduino/blob/master/examples/StandardFirmata/StandardFirmata.ino#L625-L632 This may not be relevant to all IO-Plugins, they may only need to provide the mode for compatibility and override it in `pinMode`. For Johnny-Five analog sensors to work with an IO Plugin, they need to support the conversion of 2 => 0 or 2 as it's used in firmata.

**analogWrite(pin, value)**
**pwmWrite(pin, value)** (to supercede `analogWrite`)
- Ensure pin mode is PWM (3)
- Ensure PWM capability
- Accept an 8 bit value (0-255) to write

**digitalWrite(pin, value)**
- Ensure pin mode is OUTPUT (1)
- Write HIGH/LOW (single bit: 1 or 0)

**i2cWrite(address, inBytes)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- `inBytes` is an array of data bytes to write

**i2cWrite(address, register, inBytes)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- `register` is a single byte
- `inBytes` is an array of data bytes to write

**i2cWriteReg(address, register, value)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- `register` is a single byte
- `value` is a single byte value

**servoWrite(pin, value)**
- Ensure pin mode is SERVO (4)
  - Can be transformed
- Ensure PWM capability
- Accept an 8 bit value (0-255) to write

**analogRead(pin, handler)**
- Ensure pin mode is ANALOG (2)
  - Can be transformed
- Create a `data` event stream, invoking `handler` at an implementation independent frequency, however it is recommended that `handler` is called no less than once every 19 milliseconds.
- A corresponding "analog-read-${pin}" event is also emitted

**digitalRead(pin, handler)**
- Ensure pin mode is INPUT (0)
  - Can be transformed
- Create a `data` event stream,  invoking `handler` at an implementation independent frequency, however it is recommended that `handler` is called no less than once every 19 milliseconds.
- A corresponding "digital-read-${pin}" event is also emitted

**i2cRead(address, register, bytesToRead, handler)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- `register` is set prior to, or as part of, the read request.
- Create a single `data` event,  invoking `handler` once per read, continuously and asynchronously reading.
- A corresponding "I2C-reply-${address}-${register}" event is also emitted

**i2cRead(address, bytesToRead, handler)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- Create a single `data` event,  invoking `handler` once per read, continuously and asynchronously reading.
- A corresponding "I2C-reply-${address}-0" event is also emitted

**i2cReadOnce(address, register, bytesToRead, handler)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- `register` is set prior to, or as part of, the read request.
- Create a single `data` event,  invoking `handler` once after a single asynchronous read.
- A corresponding "I2C-reply-${address}-0" event is also emitted

**i2cReadOnce(address, bytesToRead, handler)**
- Ensure pin mode is UNKNOWN (99)
  - Can be transformed
- Create a single `data` event,  invoking `handler` once after a single asynchronous read.
- A corresponding "I2C-reply-${address}-${register}" event is also emitted

**pingRead(settings, handler)**
This method is defined solely to handle the needs of `HCSR04` (and similar) components. 
- No pin mode specified
- Create a single `data` event,  invoking `handler` once per read, continuously and asynchronously reading.
- `settings` is an object that includes the following properties and corresponding values: 
  ```
  {
    pin: pin number,
    value: HIGH,
    pulseOut: 5   
  }
  ```
- `handler` is called with a duration value, in `microseconds`, which is the result of take a pulsed measurement as required by `HCSR04` (and similar) devices.
- A corresponding "I2C-reply-${address}-${register}" event is also emitted


### Special Method Definitions

**normalize(pin)**
- Define a special method that Johnny-Five will call when normalizing the pin value.
```js
// Examples: 
var io = new IOPlugin();

// The board might want to map "A*" pins to their integer value, 
// eg. Arduino allows user code to write "A0" for pin 14:
io.normalize("A0"); // 14
```


### Special Property Definitions

**defaultLed**
- This is the pin address for the board's default, built-in led.



### TODO

- Define pluggable transports, for example: replacing node-serialport with socket.io-serialport and similar. 
