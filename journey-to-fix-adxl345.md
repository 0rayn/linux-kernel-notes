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

-> why bother adding in_accell_scale ?
    => 

-> what's an event ?
     An event is a notification that something "noteworthy" happened in the
    hardware. It's usually based on a Threshold.

-> What's a triger ?
      A trigger is a "source of timing". It tlls the system when to take
     a measurement. You can have a *Timer Trigger* (take a sample each 10ms),
     or a *Data-ready Trigger* (the sensor has new data ready, take it now).
     It controls the flow of raw data into the IIO buffer.




- How to test if I did everything right without the hardware ?

- Can the raspberry pi 5 I have help here ?

- in the proccess of adding th in_accell_scale do I need to add something to
the code ?

- Do I need to remove that comment that stated that the driver uses that
value without adhearing to the ABI after I do that ?
