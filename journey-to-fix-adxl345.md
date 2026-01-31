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
-> Why shose m/s^2 ? 
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
