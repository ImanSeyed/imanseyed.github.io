---
title: "Internal of MATLAB Support Package for Arduino Hardware"
categories: [embedded]
tags: [off-topic]
---

------

Don't ask me how I ended up playing with MATLAB. That is on my university, not me.

While uploading code to an Arduino, one thing bothered me. MATLAB kept the object returned by the `arduino` constructor alive after my script finished, so I had to clear it every time. Fine. The error message was interesting though:

```
Error using arduino (line 158)
MATLAB connection to Uno at /dev/ttyUSB0 exists in your workspace. To create a new connection, clear the existing object.
```

And:

```shell
$ lsof /dev/ttyUSB0
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
MATLAB  6026 iman  741u   CHR  188,0      0t0  919 /dev/ttyUSB0
```

MATLAB is holding file descriptor 741 on `/dev/ttyUSB0` which is the serial link to the board. During setup it even said "Upload Arduino Server". So I asked myself, what does that mean? I usually live in C and C++ for embedded, so I had to dig in.

> I found the Firmata approach (for Java or other languages). MATLAB does not use Firmata though. It uses something "similar".
> - https://www.yorku.ca/professor/drsmith/2024/06/11/firmata-example-i2c-sensor-java-firmata4j/ (Written by Professor James Andrew Smith)
{: .prompt-tip}

## Investigation

Error strings are always good breadcrumbs. That message led me to `validateConnection()`, called by `initHardware()` in the `arduino` class. Keep in mind that the `arduino` class subclasses `matlabshared.hwsdk.controller`.

```matlab
classdef (Sealed) arduino < matlabshared.hwsdk.controller
...
end

classdef controller < matlabshared.hwsdk.controller_base &...
        matlab.mixin.CustomDisplay
...        
		function initHardware(obj, varargin)
		...
            % Check if transport layer already occupied, or hardware connection already exists.
            validateConnection(obj.ConnectionController, addressKey);
		...
		end
end
```

**TRANSPORT LAYER**?! Now we are getting somewhere. I turned on tracing to find out more about what is going on.

```matlab
% > help arduino
%        TraceOn - Program log and Arduino commands log
ledPin = "D4";
a = arduino("/dev/ttyUSB0", "Uno", TraceOn=true);
configurePin(a, ledPin, "DigitalInput");
```

Output:

```cpp
>> arduino_test
ArduinoTraceOn::pinMode(4, INPUT);
ArduinoTraceOn::pinMode(4, INPUT);
>> clear a
UnconfigureDigitalPin::pinMode(4, INPUT);
```

Interesting, but not enough! Time for `strace`. Assume `$SupportPackageRoot` is `MATLAB/SupportPackages/<version>/`.

```shell
$ strace -F -e openat matlab -desktop -sd /home/iman/w/matlab/ &| rg -v "arduinoio" | rg "arduino"
openat(AT_FDCWD, ".$SupportPackageRoot/toolbox/matlab/hardware/supportpackages/sharedarduino", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 62
openat(AT_FDCWD, "$SupportPackageRoot/toolbox/matlab/hardware/supportpackages/sharedarduino/+arduinodriver", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 67
openat(AT_FDCWD, "$SupportPackageRoot/toolbox/matlab/hardware/supportpackages/sharedarduino/+arduinodriver/ArduinoDigitalIO.p", O_RDONLY) = 77
openat(AT_FDCWD, "$SupportPackageRoot/toolbox/matlab/hardware/supportpackages/sharedarduino/+arduinodriver/ArduinoDigitalIO.m", O_RDONLY) = 77
...
```

Okay, don't panic. These are just the files MATLAB opens when the script runs, nothing weird is going on.

## Package Structure

```shell
$ ls $SupportPackageRoot/toolbox/matlab/hardware/supportpackages/sharedarduino--tree -L2
├── +arduinodriver # high-level MATLAB classes for Analog/Digital/I2C/SPI/Serial APIs
│  ├── ArduinoDigitalIO.m
│  ├── ArduinoDigitalIO.p
│  ├── ArduinoI2C.m
│  ├── ArduinoI2C.p
	...
├── +arduinoio 
│  ├── +internal
│  ├── CLIRoot.m # points to arduino-cli
	...
│  └── SharedArduinoRoot.m
├── +simulinkio
│	...
└── target # on-board server firmware (C/C++)
   ├── ArduinoServer.ino
   ├── server/
   └── transport/

```

> MATLAB ships many internals as p-code. It means that they are content-obscured, and you can't read them without getting your hand dirty. If a `.m` and `.p` share a name in the same directory, MATLAB runs the `.p` and uses the `.m` only for help and signatures. `pcode` hides the implementation while keeping the public API visible.
{: .prompt-tip}

## Sketch Upload Process

### What gets uploaded

When MATLAB decides it needs to reflash, it uploads `target/ArduinoServer.ino` plus the modules you enabled, for example I2C and SPI.

```
target/
  ArduinoServer.ino          % setup/loop + packet server(...)
  server/
    MW_digitalIO.cpp         % pinMode/read/write
    MW_AnalogInput.cpp       % analogRead
    MW_I2C.cpp, MW_SPI.cpp   % buses
    ...
  transport/
    rtiostream_serial_*.cpp  % serial/BLE/Wi-Fi backends
```

These use the usual Arduino core API ([Wiring framework](https://wiring.org.co/reference/)) like `pinMode` and `digitalWrite` internally.

### How it gets uploaded

MATLAB invokes `arduino-cli` from the support package.

`+arduinoio/CLIRoot.m`:

```matlab
function output = CLIRoot()
% This function returns the installed directory of Arduino CLI
%   Copyright 2023 The MathWorks, Inc.
    output = arduinoio.getArduinoCLIRootDir;
end
```

```matlab
>> arduinoio.getArduinoCLIRootDir
ans =
    "$SupportPackageRoot/3P.instrset/arduinoide.instrset/aCLI"
```

And:

```shell
$ ls $SupportPackageRoot/3P.instrset/arduinoide.instrset/aCLI/
arduino-cli  arduino-cli.yaml  data  LICENSE.txt  user
```

``arduino-cli` is the tool that compiles the sketch (using `gcc-avr` on my system) and uploads it. To verify this, watch the `arduino-cli` executable, upload a sketch from somewhere other than MATLAB, then return to MATLAB and run your script. You should see this on the Command Window:

```
Updating server code on board Uno (/dev/ttyUSB0). This may take a few minutes.
```

If you trace the MATLAB process, you can see the arguments passed to it as well:

```shell
$ strace -F -s 1000 -e execve matlab -desktop -sd /home/iman/w/matlab/ &| rg "arduino-cli"
execve("$SupportPackageRoot/3P.instrset/arduinoide.instrset/aCLI/arduino-cli", ["$SupportPackageRoot/3P.instrset/arduinoide.instrset/aCLI/arduino-cli", "-v", "--fqbn", "arduino:avr:uno", "compile", "--upload", "/tmp/iman/ArduinoServerUno", "--port", "/dev/ttyUSB0", "--build-path", "/tmp/iman/ArduinoServerUno/MW", "--config-file", "$SupportPackageRoot/3P.instrset/arduinoide.instrset/aCLI/arduino-cli.yaml"], 0x60e8f9422650 /* 70 vars */) = 0
```

In other words:

```shell
$ arduino-cli \
  -v \
  --fqbn arduino:avr:uno \
  compile \
  --upload "/tmp/iman/ArduinoServerUno" \
  --port /dev/ttyUSB0 \
  --build-path "/tmp/iman/ArduinoServerUno/MW" \
  --config-file "$SupportPackageRoot/3P.instrset/arduinoide.instrset/aCLI/arduino-cli.yaml"

```

## How MATLAB Talks to the Board

### Two execution modes behind one API

From files like `+arduinodriver/ArduinoDigitalIO.m`:

```matlab
if ~coder.target('MATLAB')
    % This is executed during MATLAB and Simulink code generation <==========
    pin = varargin{1};
    mode = varargin{2};
    coder.cinclude('MW_arduino_digitalio.h');
    coder.ceval('digitalIOSetup',uint8(pin), mode);
    
else
    % This is executed during MATLAB and Simulink IO <==========
    writeStatus = configureDigitalPinInternal@matlabshared.ioclient.peripherals.DigitalIO(obj, varargin{:});
    if(nargout == 1)
        varargout{1} = writeStatus;
    end
end
```

**Non-codegen (interactive MATLAB):** you run scripts in MATLAB on the PC like what I just did; calls like `arduino(...)`, `writeDigitalPin(...)` send requests with different IDs to a **server sketch** that MATLAB flashed onto the board. No C/C++ is *generated*.

**Codegen (generated firmware):** MATLAB (via **MATLAB Coder**) turns your code into **C/C++** that runs **standalone on the MCU** (no MATLAB, no server sketch at runtime). Same API, different backend.

*(Simulink can also generate code for Arduino; I'm not covering it here)*

### What the `arduino` object holds

It is not only a serial port. It **wraps a transport handle**, a **connection registry** that stops duplicate connections, and **the library set you selected** when the server sketch was built. That is why the serial port stays open and why you need to clear the object when you are done.

### Where your MATLAB call turns into packets (non-codegen)

Your `.m` class hands off to a p-coded superclass that builds and sends packets. In `ArduinoDigitalIO.m` you will see calls like:

```matlab
configureDigitalPinInternal@matlabshared.ioclient.peripherals.DigitalIO(...)
writeDigitalPinInternal@matlabshared.ioclient.peripherals.DigitalIO(...)
readDigitalPinInternal@matlabshared.ioclient.peripherals.DigitalIO(...)
unconfigureDigitalPinInternal@matlabshared.ioclient.peripherals.DigitalIO(...)
```

### The protocol in one glance over serial

There is no public documentation for it, but if you open the `ArduinoServer.ino` sketch you will see it includes `IO_packet.h` and related files. If you go to your MATLAB installation path, you can read them. In short, MATLAB talks to the board using a private, packetized protocol (header + payload length + CRC-8 J1850). The serial, BLE, and Wi-Fi `transport/rtiostream_*` files are only the transport; framing and CRC live in `IO_packet.*` on the target and in MATLAB’s host code. On receive, the firmware waits for the header, reads the length, validates it if CRC is enabled, then reads IDs and payload and executes. For example, if you call:
```matlab
writeDigitalPin(a, "D4", 1);
```

MATLAB packs something like this:
```
+---------------------+
| HEADER_HOSTTOTARGET |
+---------------------+
|     payloadSize     |
+---------------------+
| uniqueId            |
+---------------------+
| requestId = 0x0001  | <--- WRITEDIGITALPIN
+---------------------+
| isRawRead = 0x00    |
+---------------------+
| payload:            |
|   pinNo             |
|   value (LOW/HIGH)  |
+---------------------+
|       CRC8          |
+---------------------+
```

You can see the full list of `requestId` macros in `ioserver/inc/IO_requestID.h`. MATLAB then sends the packet over the selected transport, serial in my case. On the Arduino, the `server()` function parses it, sees `WRITEDIGITALPIN`, calls `MW_digitalIO_write`, which calls `digitalWrite` and sets the pin HIGH.

## TL;DR

- Interactive MATLAB means a server on the board and packets over serial, Wi Fi, or BLE.

- Deploy and codegen means no server, just compiled C/C++ on the MCU. 

- The `arduino` object manages the transport and the connection registry, so the port stays busy until you clear it.
