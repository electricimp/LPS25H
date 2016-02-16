# Driver for the LPS25H Air Pressure / Temperature Sensor

Author: [Tom Byrne](https://github.com/ersatzavian/)

The [LPS25H](http://www.st.com/web/en/resource/technical/document/datasheet/DM00066332.pdf) is a MEMS absolute pressure sensor. This sensor features a large functional range (260-1260hPa) and internal averaging for improved precision.

The LPS25H can interface over I&sup2;C or SPI. This class addresses only I&sup2;C for the time being.

**To add this library to your project, add** `#require "LPS25H.class.nut:2.0.1"` **to the top of your device code**

## Hardware

To use the LPS25H, connect the I2C interface to any of the imp's I2C Interfaces. To see which pins can act as an I2C interface, see the [imp pin mux](https://electricimp.com/docs/hardware/imp/pinmux/) on the Electric Imp Developer Center.

The LPS25H Interrupt Pin behavior can be configured through this class. The corresponding pin on the imp and associated callback are not configured or managed through this class. To use the interrupt pin:

- Connect the LPS25H's "INT1" pin to an imp pin
- Configure the imp pin connected to INT1 as a DIGITAL_IN with your desired callback function
- Use the methods in this class to configure the interrupt behavior as needed

![LPS25H Circuit](./circuit.png)

## Class Usage

### Constructor

The constructor takes two arguments to instantiate the class: a pre-config
ured I&sup2;C bus and the sensor’s I&sup2;C address. The I&sup2;C address is optional and defaults to 0xB8 (8-bit address).

```squirrel
const LPS25H_ADDR = 0xBA;    // non-default 8-bit I2C Address for LPS25H (SA0 pulled high)

hardware.i2c89.configure(CLOCK_SPEED_400_KHZ);
pressureSensor <- LPS25H(hardware.i2c89, LPS25H_ADDR);
```

### Reset

The LPS25H is not automatically reset when constructed so that Electric Imp applications can use the device through sleep/wake cycles without it losing state. To reset the device to a known state, call the *softReset* method:

```squirrel
hardware.i2c89.configure(CLOCK_SPEED_400_KHZ);
pressureSensor <- LPS25H(hardware.i2c89);
pressureSensor.softReset();
```

After a reset, the LPS25H will be disabled. Call *enable* to use the device.

### Class Methods


### enable(*state*)

Enable (*state* = true) or disable (*state* = false) the LPS25H. The device must be enabled before attempting to read the pressure or temperature.  If the device is disabled and a reading is taken it will return stale data.

```squirrel
pressureSensor.enable(true);    // Enable the sensor
```

### read([*callback*])

The **read()** method returns a pressure reading in hPa.  The reading result is in the form of a squirrel table with the field *pressure*.  If an error occurs during the reading the pressure field will be null, and the reading table will contain an additional field *err*, with a description of the error.

If a callback parameter is provided the reading executes asynchrounously, and the results table will be passed to the supplied function as the only parameter. If not, the reading executes synchronously, and the results table will be returned.

#####asynchrounous example:
```squirrel
pressureSensor.read(function(result) {
  if ("err" in result) {
    server.error("An Error Occurred: "+result.err);
  } else {
  	server.log(format("Current Pressure: %0.2f hPa", result.pressure));
  }
});
```
#####synchrounous example:
```squirrel
function hpaToHg(hpa) {
  return (1.0 * hpa) / 33.8638866667;
}

local result = pressureSensor.read();

if ("err" in result) {
  server.error("An Error Occurred: "+result.err);
} else {
  server.log(format("Current Pressure: %0.2f in. Hg", hpaToHg(result.pressure));
}
```

### getTemp()

Returns temperature in degrees Celsius.

```squirrel
server.log("Current Temperature: "+pressure.getTemp() + " C");
```

### setReferencePressure(*pressure*)

Set the reference pressure for differential pressure measurements and interrupts (see *configureInterrupt*). Reference Pressure is in hectopascals (hPa). Negative pressures are supported. Reference Pressure range is +/- 2046 hPa.

```squirrel
server.log("Internal Reference Pressure Offset = " + pressure.getReferencePressure());
```

### getReferencePressure()

Get the reference pressure for differential pressure measurements and interrupts (see *configureInterrupt*). Reference Pressure is in hectopascals (hPa).

```squirrel
server.log("Internal Reference Pressure Offset = " + pressure.getReferencePressure());
```

###setDataRate(*dataRate*)

Sets the output data rate (ODR) of the pressure sensor in Hz. The nearest supported data rate less than or equal to the requested rate will be used and returned. Supported datarates are 0 (one shot configuration), 1, 7, 12.5, and 25 Hz. The default value is 0 Hz.

```squirrel
local dataRate = pressureSensor.setDataRate(7);
server.log(dataRate);
```

###getDataRate()

Returns the output data rate (ODR) of the pressure sensor in Hz.

```squirrel
local dataRate = pressureSensor.getDataRate();
server.log(dataRate);
```

### configureInterrupt(*enable*, [*threshold*, *options*])

This method configures the interrupt pin driver, threshold, and sources.  The device starts with this disabled by default.

####Parameters

| Parameter | Type | Default | Description |
| -------------- | ------ | ---------- | --------------- |
| *enable* | boolean | N/A | Set true to enable the interrupt pin. |
| *threshold* | Inteter | null | Interrupts are generated on differential pressure events; a high differential pressure interrupt occurs if (Absolute Pressure - Reference Pressure) > Threshold, a low differential pressure interrupt occurs if (Absolute Pressure - Reference Pressure) < (-1.0 * Threshold). The threshold is expressed in hectopascals (hPa). |
| *options* | Byte | 0x00 | Configuration options combined with the bitwise OR operator. See the ‘Options’ table below. |

####Options

| Option Constant | Description |
| -------- | ----------- |
| INT_ACTIVELOW | Interrupt pin is active-high by default. Use to set interrupt to active-low. |
| INT_OPENDRAIN | Interrupt pin driver push-pull by default. Use to set interrupt to open-drain. |
| INT_LATCH | Interrupt latching mode is disabled by default.  Use to enable interrupt latching mode.  To clear a latched interrupt pin call getInterruptSrc() |
| INT_LOW_PRESSURE | Interrupt is disabled by default.  Use to enable interrupt when pressure below threshold |
| INT_HIGH_PRESSURE | Interrupt is disabled by default.  Use to enable interrupt when pressure above threshold |

```squirrel
// Enable interrupt, configure as push-pull, active-high, latched. Fire interrupt if (absolute pressure - reference pressure) > 10 hPa
pressureSensor.configureInterrupt(true, 10, LPS25H.INT_LATCH | LPS25H.INT_HIGH_PRESSURE);
```

```squirrel
// Enable interrupt, configure as open-drain, active-low, latched. Fire interrupt if (absolute pressure - reference pressure) < -20 hPa
pressureSensor.configureInterrupt(true, 20, LPS25H.INT_ACTIVELOW | LPS25H.INT_OPENDRAIN | LPS25H.INT_LATCH | LPS25H.INT_LOW_PRESSURE);
```

### getInterruptSrc()

Determine what caused an interrupt, and clear latched interrupt. This method returns a table with three keys to provide information about which interrupts are active.

| key | Description | Notes |
| -------- | ----------- | ----- |
| int_active | *true* if an interrupt is currently active or latched | |
| high_pressure | *true* if the active or latched interrupt was due to a high pressure event | |
| low_pressure | *true* if the active or latched interrupt was due to a low pressure event | |

```squirrel
// Check the interrupt source and clear the latched interrupt
local intSrc = pressureSensor.getInterruptSrc();
if (intSrc.int_active) {
  // interrupt is active
  if (intSrc.high_pressure) {
    server.log("High Pressure Interrupt Occurred!");
  }
  if (intSrc.low_pressure) {
    server.log("Low Pressure Interrupt Occurred!");
  }
} else {
  server.log("No Interrupts Active");
}
```

### setPressNpts(*numberOfReadings*)

Set the number of readings taken and then internally averaged to produce a pressure result. The value provided will be rounded up to the nearest valid value: 8, 32 ,128 or 512. The actual value used is returned.

```squirrel
// Fastest readings, lowest precision
pressureSensor.setPressNpts(8);

// Slowest readings, highest precision
pressureSensor.setPressNpts(512);

// Rounding and checking result
local actualNpts = pressureSensor.setPressNpts(30);
// prints "Actual Pressure Npts = 32"
server.log("Actual Pressure Npts = " + actualNpts);
```

### setTempNpts(*numberOfReadings*)

Set the number of readings taken and internally averaged to produce a temperature result. The value provided will be rounded up to the nearest valid value: 8, 16, 32 and 64. The actual value used is returned.

```squirrel
// Fastest readings, lowest precision
pressureSensor.setTempNpts(8);

// Slowest readings, highest precision
pressureSensor.setTempNpts(64);
```


### softReset()

Reset the LPS25H from software. Device will come up disabled.

```squirrel
pressureSensor.softReset();
```

### getDeviceID()

Returns the value of the device ID register, BDh.

## License

The LPS25H library is licensed under the [MIT License](./LICENSE).
