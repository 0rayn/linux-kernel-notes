# Industrial IO
## What's IIO:
  The main purpose of the Industrial I/O subsystem (IIO) is to 
  provide support for devices that in some sense perform either 
  analog-to-digital conversion (ADC) or digital-to-analog conversion (DAC)
  or both.
  Things like:
    - ADCs
    - DACs
    - Gyroscopes
    - proximity sensors
    ...
## Core elements:
### Industrial I/O Devices:
- An IIO device usually corresponds to a single hardware sensor and it provides all the information needed by a driver handling a device.
  // TODO: learn why they said "usually corresponds to a single hardware":
    => Multi-function devices: some devices can have multiple functionalities like the nvenSense MPU-6050 that contains
  both a 3-axis accelerometer and a 3-axis gyroscope. In this case this driver may register as
  two different devices.
    => Composite logic devices: you may have two sensors register as one device so even if you're pulling data from 
    two different chips the user only see one device giving full data (humidity and temperature for example).
    => high-density boards: we might have a single chip that acts as an ADC with 16 or 32 channels,
    while it's one chip, it might be controlling 32 entirely different sensors.

- There are two ways for a user space application to interact with an IIO driver.
  - /sys/bus/iio/iio:deviceX/, this represents a hardware sensor and groups together the data channels of the same chip.
  - /dev/iio:deviceX, character device node interface used for buffered data transfer and for events information retrieval.
  // TODO: What are even these 'channels':
    => A channel represents a single steam of data originating from the sensor.
    => Like probing with a logic analyzer each data pin (in voltage, accelerometer_x, temperature input).
    => every channel is defined in the driver using struct iio_chan_spec:
      => type: what kind of data is this? (voltage, Acceleration ...)
      => Index/Modifier: is this "X" axis or the "Y" axis? or is it "channel 3" of an ADC.
      => Info Mask: What can the user do with this? (Read a raw value, read a scale factor, or set an offset).

#### What to do at probe:
  // TODO: What's probing ??
    => probing is the proccess of matching the drivers with the devices (detected via the device
  teee or ACPI).
    == How it works:
      1- registration: when the IIO Module loads it tell the kernel "I'm a driver for a
    sensor named MPU-6050".
      2- matching: the kernel looks at the hardware bus (I2C, SPI, etc). If it finds a hardware
    node with the compatible string `MPU-6050`, it says, "we have a match"
      3- The Probe call: the kernel then executes the probe function defined in the driver's
    code.
    == this is just an abstraction of what happens I have to learn more.


1- Call iio_device_alloc(), which allocates memory for an IIO device.
2- Initialize IIO device fields with driver specific information (e.g. device name, device channels).
3- Call iio_device_register(), this registers the device with the IIO core. After this call the device is ready to accept requests from user space applications.
#### What to do at remove:
1- iio_device_unregister(), unregister the device from the IIO core.
2- iio_device_free(), free the memory allocated for the IIO device.

### IIO device sysfts interface:

- Attributes are sysfs files used to expose chip info and also allowing applications to set various configuration parameter.
more at: https://www.kernel.org/doc/html/v6.9/driver-api/iio/core.html#iio-device-sysfs-interface

  `sysfs` is an in-memory filesystem that allows the kernel to export internal data structures (and their settings) to user space as simple text files

### IIO device channels:
  struct iio_chan_spec - specification of a single channel
- Already did my TODO before about this but here are some examples:
  - a thermometer sensor has one channel representing the temperature measurement.
  - a light sensor with two channels indicating the measurements in the visible and infrared spectrum.
  - an accelerometer can have up to 3 channels representing acceleration on X, Y and Z axes.
#### masking is brilliant:
oh lovely so those channels are what can be exposed to the user via the sysfs, and when defining
channel specs we use bitmasks (info_mask) for example:
    - info_mask_separate: this is used for the actual data, for example temperature in.
    - info_mask_shared_by_all: use this for somehting like ht esampling_frequency
    of the chip, if that changes it usually changes for every sensor on that chip.
#### Modified vs indedex:
when there are multiple data channels per channel type we have two ways
to distinguish between them:
- Use Modifiers (.modified = 1) when the channels are physically different.
  Example: An accelerometer has 3 axes (X, Y, Z). They are all "acceleration," but they represent different physical dimensions.
  Result: in_accel_x_raw, in_accel_y_raw.

- Use Indexing (.indexed = 1) when the channels are identical copies.
  Example: An ADC (Analog-to-Digital Converter) has 8 pins. They all do the exact same thing; they are just numbered.
  Result: in_voltage0_raw, in_voltage1_raw.
#### Proccessed vs Raw:
- RAW: This is the "naked" integer from the hardware register (e.g., 4095 for a 12-bit ADC). To make sense of it, user-space needs the _scale attribute to convert it to Volts or Celsius.

- PROCESSED: The driver does the math for you. When you cat the file, you get a value that is already converted to standard units (multiplied by 1000, usually, to avoid floating point in the kernel).

example:

```
```
```
```
```
```
static const struct iio_chan_spec light_channels[] = {
        {
                .type = IIO_INTENSITY,
                .modified = 1,
                .channel2 = IIO_MOD_LIGHT_IR,
                .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
                .info_mask_shared = BIT(IIO_CHAN_INFO_SAMP_FREQ),
        },
        {
                .type = IIO_INTENSITY,
                .modified = 1,
                .channel2 = IIO_MOD_LIGHT_BOTH,
                .info_mask_separate = BIT(IIO_CHAN_INFO_RAW),
                .info_mask_shared = BIT(IIO_CHAN_INFO_SAMP_FREQ),
        },
        {
                .type = IIO_LIGHT,
                .info_mask_separate = BIT(IIO_CHAN_INFO_PROCESSED),
                .info_mask_shared = BIT(IIO_CHAN_INFO_SAMP_FREQ),
        },
   }
```
```
```
```
```
```

This channel’s definition will generate two separate sysfs files for raw data retrieval:
  /sys/bus/iio/iio:deviceX/in_intensity_ir_raw
  /sys/bus/iio/iio:deviceX/in_intensity_both_raw
one file for processed data:
  /sys/bus/iio/iio:deviceX/in_illuminance_input
and one shared sysfs file for sampling frequency:
  /sys/bus/iio/iio:deviceX/sampling_frequency.


