- looks like I'll work on the adxl345 first hhhhhhh
https://lore.kernel.org/linux-iio/20260129173337.7fc2a4ad@jic23-huawei/#t

- I just was looking at the familiar adxl345 and found something that
didn't make sense docs saying 62.5 g/LSB for the threashold and consulting
the documentation I saw 62.5 mg/LSB instead, so I fixed that with som more fixes
to the grammar and readablity.

- this actually made Jonathan Cameron spot even bigger issues in the documentation
and the driver.

- I can't believe when I answered I forgot to say thanks :). I really wanted to
thank his detailed feedback, based on his feedback I got some really good
resources and can plan the V2.

- Currently I need to to a patch series like this:
 => Patch 1 (cleanup): where I'll re-apply the grammar, typo, and 
    readability fixes.
 => Patch 2 (technical doc fix): I'll fix scale values and update the tables
    to have m/s^2 instead of mg.
 => patch 3 (driver fix): implement the in_accell_scale attribute to make
    the driver compliant with the ABI.

---------------- Resources and mental notes -------------------- 
-> Why chose m/s^2 ? 
    => The IIO subsystem mandates SI units for consistency so all userspace
    applications would know how to interact with the driver even without
    consulting the docs.

-> What's LSB exactly ?
    => LSB stands for Least Significant Bit. It's the smallest unit of
    change a digital sensor can detect/represent.
    => If we have an 8 bit ADC measuring a voltage from 0 to 5V, it will have
    2^8 = 256 possible steps, and the LSB value is the size of one step
    (5V/256 ≈ 19.5 mV). You can never measure 10 mV
    because it's smaller than 1 LSB.
    ------- ADXL345 ---------
    => Raw Data: When measuring constant movement, the LSB changes based on
    the range we select (In ±2g range: 1 LSB ≈ 4 mg)
    => Event Thresholds: For things like "Tap Detection" the hardware uses a
    fixed step: 1LSB = 62.5 mg (you can use this to triger an event only if a
    tap is stronger than 500mg for that we need to write 8 into th THRESH_TAP
    register. (500mg/62.5mg/LSB = 8 LSB).

-> why bother adding events/in_accel_scale ?
    => The linux kernel follows a very simple math rule:
    Physical Value (m/s^2) = (raw value + offset) * scale.
    most accelerometers including the ADXL345 have an offset of 0
    so the formula becomes:
    Physical Value = Raw value * scale.
    => by providing in_accel_scale the user can for example:
        - read from in_accel_z_raw -> for example returns 256 LSB
        - read from in_accel_scale -> for example returns 0.038312.
        - 256 * 0.038312 = 9.807872 m/s^2
    => and because the adxl345 weirdly doesn't adjust the events LSB dynamicly
    when changing the measurement range (e.g +-2g to +-16g). The regular data
    might have a scale of 0.004, but the chock detection always has a scale
    of 0.612915 resulting in:
    in_accel_scale representing on LSB of movement.
    events/in_accel_scale representing one LSB of a shock/tap.

-> what's an event ?
     An event is a notification that something "noteworthy" happened in the
    hardware. It's usually based on a Threshold.

-> What's a triger ?
      A trigger is a "source of timing". It tlls the system when to take
     a measurement. You can have a *Timer Trigger* (take a sample each 10ms),
     or a *Data-ready Trigger* (the sensor has new data ready, take it now).
     It controls the flow of raw data into the IIO buffer.

- How to test if I did everything right without the hardware ?
     Since I don't have the hardware I think I must tell them that with my
    patch, but do a compile test and checkpatch to make sure it passes the
    bare-min also I'll use the IIO Dummy to check it's code and results.

- Can the raspberry pi 5 I have help here ?
    not much.

- in the proccess of adding th in_accel_scale do I need to add something to
the code ?
     looks like I have to add a mask to the channel definition, I need to add
    the IIO_EV_INFO_SCALE to the event_spec masks. This is the signal
    that tells the IIO core to create the events/in_accel_scale

- Do I need to remove that comment that stated that the driver uses that
value without adhearing to the ABI after I do that ?
     Yes after I add the in_accel_scale we'll get the best of both worlds,
    if you're only interested by the signal intensity you can use the file
    normally, but if you want the absolute measured value you'll multiply
    that value by the value in in_accel_scale.


============================= Submited V2 =========================================
After the submition of the V2 patch I got some feedbacks:
1- Feedback from David Lechner: "I think this could be worded better. The
  enum member isn't really 'missing'... Are there actually any users of
  these attributes...?"
  => so I have to make sure to bring examples of drivers using this event
  scaling but atchieves that in a "hacky way".
  ```
```

My understanding is that making the attributes in the sysfs manually using
the IIO_DEVICE_ATTR is a hacky way when we have the IIO core making that
for us automaticaly and without the need to be worring if we do a typo in
the file name we're exposing the attribute through.

https://docs.kernel.org/driver-api/driver-model/device.html

While searching for how the other drivers do the scaling without having that
enum member I used grep and decided to go with the drivers/iio/accel/mma8452.c
as an example

```sh
```
```sh
grep -rn "IIO_DEVICE_ATTR" drivers/iio/ | grep -iE "scale"
```

```
```C

/*
 * Threshold is configured in fixed 8G/127 steps regardless of
 * currently selected scale for measurement.
 */
static IIO_CONST_ATTR_NAMED(accel_transient_scale, in_accel_scale, "0.617742");
```

so the driver had to expose a scale for the transient detection
("transient" refers to a specific type of motion detection that looks for rapid,
short-term changes in acceleration rather than a sustained level of force.)

and because there is no SCALE bit and the value is constant he had to 
expose the value manually using IIO_CONST_ATTR_NAMED.

my take on adding the IIO_EV_INFO_SCALE is to have a standard way for drivers to
expose scale of their events without having to make their own boilerplate each time
making both the process of writing drivers and editing them more straight forward
and standarazed.

I did reply to David lechner with this and he did reply that we better wait Jonathan
to see if this is the right action to make, cause if only 2 drivers uses that no need
to change the IIO core.

I do believe this is a mistake from my side next time and now I have to look more and
check if any other drivers do this. The more evidence I have the better. I didn't take
enough time to search for more examples of this.

also I did thank both David and randy for the other comments and I must do them in the V3

for now I'll take this time as an apportunity to learn more about the other drivers and
the drivers structure :) I'm new to this overall. But happy with the progress so far.

-> Waiting for Jonathan's review I'll start looking for more drivers that do the same 'hacks'
for 2 reasons: R1 - Jonathan may ask me to research more; R2 - if I heard nothing more I'll use
this as a ping with more details.

Examples of the hacks around not having the IIO_EV_INFO_SCALE in the enum iio_event_info:
1- our beloved mma8452 I already used it in my last reply to David
```C
static IIO_CONST_ATTR_NAMED(accel_transient_scale, in_accel_scale, "0.617742");
```
2- 





-------------------- I want to hightlight these while I'm on it ---------------------
```sh
grep -rn "\.event_attrs" drivers/iio/
drivers/iio/adc/ad4062.c:1311:	.event_attrs = &ad4062_event_attribute_group,
drivers/iio/adc/max1363.c:1046:	.event_attrs = &max1363_event_attribute_group,
drivers/iio/adc/ad799x.c:555:	.event_attrs = &ad799x_event_attrs_group,
drivers/iio/accel/bma400_core.c:1599:	.event_attrs = &bma400_event_attribute_group,
drivers/iio/accel/mma8452.c:1438:	.event_attrs = &mma8452_event_attribute_group,
drivers/iio/accel/adxl380.c:1660:	.event_attrs = &adxl380_event_attribute_group,
drivers/iio/imu/bmi270/bmi270_core.c:1263:	.event_attrs = &bmi270_event_attribute_group,
drivers/iio/imu/bmi323/bmi323_core.c:1822:	.event_attrs = &bmi323_event_attribute_group,
drivers/iio/dac/ad5504.c:232:	.event_attrs = &ad5504_ev_attribute_group,
drivers/iio/resolver/ad2s1210.c:1353:	.event_attrs = &ad2s1210_event_attribute_group,
drivers/iio/cdc/ad7150.c:527:	.event_attrs = &ad7150_event_attribute_group,
drivers/iio/light/apds9306.c:1151:	.event_attrs = &apds9306_event_attr_group,
drivers/iio/light/tsl2591.c:1004:	.event_attrs = &tsl2591_event_attribute_group,
drivers/iio/light/lm3533-als.c:825:	.event_attrs	= &lm3533_als_event_attribute_group,
drivers/iio/light/veml6030.c:846:	.event_attrs = &veml6030_event_attr_group,
```

while checking on the scale attribute I went to check everything else in an attempt to see
if other drivers are suffering from it or from something else entirely.

```C
static struct attribute *bmi323_event_attributes[] = {
	&iio_const_attr_in_accel_gesture_tap_value_available.dev_attr.attr,
	&iio_const_attr_in_accel_gesture_tap_reset_timeout_available.dev_attr.attr,
	&iio_const_attr_in_accel_gesture_doubletap_tap2_min_delay_available.dev_attr.attr,
	&iio_const_attr_in_accel_gesture_tap_wait_dur_available.dev_attr.attr,
	&iio_dev_attr_in_accel_gesture_tap_wait_timeout.dev_attr.attr,
	&iio_dev_attr_in_accel_gesture_tap_wait_dur.dev_attr.attr,
	&iio_const_attr_in_accel_mag_value_available.dev_attr.attr,
	&iio_const_attr_in_accel_mag_period_available.dev_attr.attr,
	&iio_const_attr_in_accel_mag_hysteresis_available.dev_attr.attr,
	NULL
};
```

```C
static struct attribute *bmi270_event_attributes[] = {
	&iio_dev_attr_in_accel_value_available.dev_attr.attr,
	&iio_const_attr_in_accel_period_available.dev_attr.attr,
	NULL
};
```

```C
static struct attribute *ad4062_event_attributes[] = {
	&iio_dev_attr_sampling_frequency.dev_attr.attr,
	&iio_dev_attr_sampling_frequency_available.dev_attr.attr,
	NULL
};
```

```C
static struct attribute *max1363_event_attributes[] = {
	&iio_dev_attr_sampling_frequency.dev_attr.attr,
	&iio_const_attr_sampling_frequency_available.dev_attr.attr,
	NULL,
};
```

```C
static struct attribute *ad799x_event_attributes[] = {
	&iio_dev_attr_sampling_frequency.dev_attr.attr,
	&iio_const_attr_sampling_frequency_available.dev_attr.attr,
	NULL,
};
```

```C
static struct attribute *bma400_event_attributes[] = {
	&iio_const_attr_in_accel_gesture_tap_value_available.dev_attr.attr,
	&iio_const_attr_in_accel_gesture_tap_reset_timeout_available.dev_attr.attr,
	&iio_const_attr_in_accel_gesture_tap_maxtomin_time_available.dev_attr.attr,
	&iio_const_attr_in_accel_gesture_doubletap_tap2_min_delay_available.dev_attr.attr,
	&iio_dev_attr_in_accel_gesture_tap_maxtomin_time.dev_attr.attr,
	NULL
};
```

```C
static struct attribute *adxl380_event_attributes[] = {
	&iio_dev_attr_in_accel_gesture_tap_maxtomin_time.dev_attr.attr,
	NULL
};
```

```C
static struct attribute *ad5504_ev_attributes[] = {
	&iio_const_attr_temp0_thresh_rising_value.dev_attr.attr,
	&iio_const_attr_temp0_thresh_rising_en.dev_attr.attr,
	NULL,
};
```

```C

static struct attribute *ad2s1210_event_attributes[] = {
	&iio_const_attr_in_phase0_mag_rising_value_available.dev_attr.attr,
	&iio_const_attr_in_altvoltage0_thresh_falling_value_available.dev_attr.attr,
	&iio_const_attr_in_altvoltage0_thresh_rising_value_available.dev_attr.attr,
	&iio_const_attr_in_altvoltage0_mag_rising_value_available.dev_attr.attr,
	&iio_dev_attr_in_altvoltage0_mag_rising_reset_max.dev_attr.attr,
	&iio_const_attr_in_altvoltage0_mag_rising_reset_max_available.dev_attr.attr,
	&iio_dev_attr_in_altvoltage0_mag_rising_reset_min.dev_attr.attr,
	&iio_const_attr_in_altvoltage0_mag_rising_reset_min_available.dev_attr.attr,
	&iio_dev_attr_in_angl1_thresh_rising_value_available.dev_attr.attr,
	&iio_dev_attr_in_angl1_thresh_rising_hysteresis_available.dev_attr.attr,
	NULL,
};
```

```C
static struct attribute *ad7150_event_attributes[] = {
	&iio_const_attr_in_capacitance_thresh_adaptive_timeout_available
	.dev_attr.attr,
	NULL,
};
```

```C
static struct attribute *apds9306_event_attributes[] = {
	&iio_const_attr_thresh_either_period_available.dev_attr.attr,
	&iio_const_attr_thresh_adaptive_either_values_available.dev_attr.attr,
	NULL
};
```

```C
static struct attribute *tsl2591_event_attrs_ctrl[] = {
	&iio_dev_attr_tsl2591_in_illuminance_period_available.dev_attr.attr,
	NULL
};
```

```C

static struct attribute *lm3533_als_event_attributes[] = {
	&dev_attr_in_illuminance0_thresh_either_en.attr,
	&lm3533_als_attr_in_illuminance0_thresh0_falling_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh0_hysteresis.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh0_raising_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh1_falling_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh1_hysteresis.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh1_raising_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh2_falling_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh2_hysteresis.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh2_raising_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh3_falling_value.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh3_hysteresis.dev_attr.attr,
	&lm3533_als_attr_in_illuminance0_thresh3_raising_value.dev_attr.attr,
	NULL
};
```

```C

static struct attribute *veml6030_event_attributes[] = {
	&iio_dev_attr_in_illuminance_period_available.dev_attr.attr,
	NULL
};
```


29 -> Available ? and I couldn't find it in the enum
```
BMI323: 7 entries
AD2S1210: 7 entries
BMA400: 4 entries
BMI270: 1 entry
AD4062: 1 entry
MAX1363: 1 entry
AD799x: 1 entry
AD7150: 1 entry
APDS9306: 1 entry
```

Well looks like we have a bigger problem than the scale bitmask hhhhhhh.
This won't serve as a ping I think I must send an RFC about it :).
1- What's the year these drivers were written in ? I want to know how long this been going,
and whither or not expect new drivers to have this in the events.
2- In the RFC I should point to these data points and entries. And ask about implementing this.
3- If it got accepted I should be ready to fix them all to use that bit hhhhhhh a lot of work
