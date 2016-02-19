# IO Plugins

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/rwaldron/io-plugins?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

(This document is a work in progress)

An IO Plugin is any class whose instances implement a [Firmata](https://github.com/jgautier/firmata) compatible interface. 
For an in-depth case study of creating an IO plugin, [read about the design and creation of the Galileo-IO Plugin here](http://bocoup.com/weblog/intel-galileo-javascript-nodejs/).

IO Plugins are used extensively by [Johnny-Five](https://github.com/rwaldron/johnny-five) to communicate with non-Arduino based hardware but may also be used independently if desired.

## Available IO Plugins

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

## Minimum Plugin Class Requirements

The plugin must...

- Be a constructor function that defines a prototype object
- Be a subclass of [EventEmitter](http://nodejs.org/api/events.html#events_class_events_eventemitter)
- Initialize instances that must...
    - asynchronously emit a "connect" event when the connection to a physical device has been made.
    - asynchronously emit a "ready" event when the handshake to the physical device is complete.
    - include a property named `isReady` whose initial value is `false`. `isReady` must be set to `true` in the same or previous execution turn as the the "ready" event is emitted.
        - The process of establishing a connection and becoming "ready" is irrelevant to this document's purposes.
    - include a readonly property named `MODES` whose value is a frozen object containing the following property/values: `{ INPUT: 0, OUTPUT: 1, ANALOG: 2, PWM: 3, SERVO: 4 }` 
    - include a readonly property named `SERIAL_PORT_IDs` whose value is a frozen object containing key/value pairs that represent a platform independent serial port identification. 
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

## Minimum API Requirements

#### pinMode(pin, mode)

- Set the mode of a specified pin to one of: 
```
INPUT: 0
OUTPUT: 1
ANALOG: 2
PWM: 3
SERVO: 4
```


### Writing

Data writing operations must be executed in order of instruction.

#### analogWrite(pin, value)

#### pwmWrite(pin, value) (to supercede `analogWrite`)

- Accept an 8 bit `value` (0-255) that is written to the specified `pin`.

#### digitalWrite(pin, value)

- Accept a `value` (0 or 1) that is written to the specified `pin`.

#### i2cWrite(address, inBytes)

- Write the array of `inBytes` to the specified `address`.

#### i2cWrite(address, register, inBytes)

- Write the array of `inBytes` to the specified `address` and `register`.

#### i2cWriteReg(address, register, value)

- Write the single `value` to the specified `address` and `register`.

#### serialWrite(portId, inBytes)

- Write the array of `inBytes` to the specified `portId`.

#### servoWrite(pin, value)

- Accept a `value` in degrees (0-180) that is written to the specified `pin`.

### Reading

All new data read processes must be asynchronous. The following methods must not block the execution process, by any means necessary. The following methods must not return the value of the new data read process.

#### analogRead(pin, handler)

- Initiate a new data reading process for `pin`
- The recommended new data reading frequency is greater than or equal to 200Hz. Read cycles may reduce to 50Hz per platform capability, but no less.
- Invoke `handler` for all new data reads, with a single argument which is the present `value` read from the `pin`.
- A corresponding `analog-read-${pin}` event is created and emitted for all new data reads, with a single argument which is the present `value` read from the `pin` (This can be used to invoke `handler`).

#### digitalRead(pin, handler)

- Initiate a new data reading process for `pin`
- The recommended new data reading frequency is greater than or equal to 200Hz. Read cycles may reduce to 50Hz per platform capability, but no less. 
- Invoke `handler` for all new data reads in which the data has changed from the previous data, with a single argument which is the present `value` read from the `pin`.
- A corresponding `digital-read-${pin}` event is created and emitted for all new data reads in which the data has changed from the previous data, with a single argument which is the present `value` read from the `pin` (This can be used to invoke `handler`).

#### i2cRead(address, register, bytesToRead, handler)

- Initiate a new data reading process for `address` and `register`, requesting the specified number of `bytesToRead`. 
- The recommended new data reading frequency is greater than or equal to 100Hz. Read cycles may reduce to 50Hz per platform capability, but no less. 
- Invoke `handler` for all new data reads, with a single argument which is an array containing the bytes read.
- A corresponding `i2c-reply-${address}-${register}` event is created and emitted for all new data reads, with a single argument which is an array containing the bytes read. (This can be used to invoke `handler`).

#### i2cRead(address, register, bytesToRead, handler)

- Initiate a new data reading process for `address`, requesting the specified number of `bytesToRead`. 
- The recommended new data reading frequency is greater than or equal to 100Hz. Read cycles may reduce to 50Hz per platform capability, but no less. 
- Invoke `handler` for all new data reads, with a single argument which is an array containing the bytes read.
- A corresponding `i2c-reply-${address}` event is created and emitted for all new data reads, with a single argument which is an array containing the bytes read. (This can be used to invoke `handler`).

#### i2cReadOnce(address, register, bytesToRead, handler)

- Initiate a new data reading process for `address` and `register`, for the specified number of `bytesToRead`. 
- This new data read occurs only once. 
- Invoke `handler` with a single argument which is an array containing the bytes read.
- A corresponding `i2c-reply-${address}-${register}` "once" event is created and emitted, with a single argument which is an array containing the bytes read. (This can be used to invoke `handler`).

#### i2cReadOnce(address, bytesToRead, handler)

- Initiate a new data reading process for `address`, requesting the specified number of `bytesToRead`. 
- This new data read occurs only once. 
- Invoke `handler` for all new data reads, with a single argument which is an array containing the bytes read.
- A corresponding `i2c-reply-${address}` event is created and emitted, with a single argument which is an array containing the bytes read. (This can be used to invoke `handler`).

#### pingRead(settings, handler)

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


#### serialRead(portId, handler)
#### serialRead(portId[, maxBytesToRead], handler)

- Initiate a new data reading process for `portId`, optionally capping the read to the specified `maxBytesToRead`. 
- The recommended new data reading frequency is greater than or equal to 100Hz. Read cycles may reduce to 50Hz per platform capability, but no less.
- Invoke `handler` for all new data reads, with a single argument which is an array containing the bytes read.
- A corresponding `serial-data-${portId}` event is created and emitted for all new data reads, with a single argument which is an array containing the bytes read. (This can be used to invoke `handler`).


### Configuring

#### i2cConfig(options)

- `options` is an object that MUST include, at very least, the following properties and corresponding values: 
  
  | Property | Description |
  |----------|-------------|
  | address  | The address of the I2C component |

- `options` _may_ include any of the common configuration properties: 
  
  | Property | Description |
  |----------|-------------|
  | bus      | The I2C bus |
  | port     | The I2C bus port |
  | delay    | Time µs between setting a register and reading the bus |

- **i2cConfig** will always be called once by every I2C component controller class in Johnny-Five. This means that any setup necessary for a given platform's I2C peripheral capabilities should be done here. 
  + Examples: 
    * Firmata.js will negotiate a default register read in µs and a default value for the `stopTX` flag.
    * Tessel-IO will negotiate the `bus` to use.
    * Tessel-IO will negotiate the `bus` to use.
- All options specified by a user program in the instantiation of a component will be forwarded to **i2cConfig**. 


#### serialConfig(options)

- Must be called to configure a serial/uart port
- `options` is an object that MUST include, at very least, the following properties and corresponding values: 

  | Property | Description |
  |----------|-------------|
  | portId   | Some value that identifies the serial/uart port to configure |

- `options` _may_ include any of the common configuration properties: 
  
  | Property | Description |
  |----------|-------------|
  | baud     | The baud rate |
  | rxPin    | The RX pin |
  | txPin    | The TX pin |

All options specified by a user program in the instantiation of a component will be forwarded to **serialConfig**. 


#### servoConfig(options)

- May be called as an alternative to calling `pinMode(pin, SERVO)`.
- `options` is an object that MUST include, at very least, the following properties and corresponding values: 
  
  | Property | Description |
  |----------|-------------|
  | pin      | The pin number/name attached to the servo |
  | min      | The minimum PWM pulse time in microseconds |
  | max      | The maximum PWM pulse time in microseconds |


#### servoConfig(pin, min, max)

- See #servoConfig(options)



### IO Control

These additions are still pending.

#### serialStop(portId)

- Stop continuous reading of the specified serial `portId`. 
- This does not close the port, it stops reading it but keeps the port open.

#### serialClose(portId)

- Close the specified serial `portId`.

#### serialFlush(portId)

- Flush the specified serial `portId`. For hardware serial, this waits for the transmission of outgoing serial data to complete. For software serial, this removes any buffered incoming serial data.





### Special Method Definitions

#### normalize(pin)

- Define a special method that Johnny-Five will call when normalizing the pin value.
```js
// Examples: 
var io = new IOPlugin();

// The board might want to map "A*" pins to their integer value, 
// eg. Arduino allows user code to write "A0" for pin 14:
io.normalize("A0"); // 14
```


### Special Property Definitions

#### defaultLed

- This is the pin address for the board's default, built-in led.



### TODO

- Define pluggable transports, for example: replacing node-serialport with socket.io-serialport and similar. 
