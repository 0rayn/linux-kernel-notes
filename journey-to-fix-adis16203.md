I started by finding the file in the staging directory.
found out a warning using:
```
./scripts/checkpatch.pl -f drivers/staging/iio/accdel/adis16203.c
WARNING: DT compatible string "adi,adis16203" appears un-documented -- check ./Documentation/devicetree/bindings/
#296: FILE: drivers/staging/iio/accel/adis16203.c:296:
+	{ .compatible = "adi,adis16203" },

total: 0 errors, 1 warnings, 0 checks, 315 lines checked

NOTE: For some of the reported defects, checkpatch may be able to
      mechanically convert to the typical style using --fix or --fix-inplace.

drivers/staging/iio/accel/adis16203.c has style problems, please review.

NOTE: If any of the errors are false positives, please report
      them to the maintainer, see CHECKPATCH in MAINTAINERS.
```
My first instinct was to make the documentation neccecary for it.
but after finding:
```
ls ./Documentation/devicetree/bindings/iio/accel/adi,adis162*
./Documentation/devicetree/bindings/iio/accel/adi,adis16201.yaml
./Documentation/devicetree/bindings/iio/accel/adi,adis16240.yaml
```
I found inside adi,adis16201.yaml both 16201 and 16209 so maybe all we need is to add 16203 if they're
very similar.

but I found after digging in the linux lore:
https://lore.kernel.org/linux-staging/20250308144239.0442f1a7@jic23-huawei/T/#t
https://lore.kernel.org/linux-iio/20230124094450.0000272b@Huawei.com/

seems like the adis16203 is too similar to the adis16201 that it doesn't need a separate driver
so merging them both into one is the correct move to do.
==> also Jonathan Cameron already tried doing that in 2023
https://lore.kernel.org/linux-iio/20230123211758.563383-13-jic23@kernel.org/
so it did end with just writing the binding
https://lore.kernel.org/linux-iio/CACRpkdZrgLQsR-BkT-FG4HvJv3fvQ1ETRcaJYAXahOcM5mcYOw@mail.gmail.com/

in an attempt for fixing the driver

so let's go into the merging
to verify this more I'll go through the datasheets throughly and find the diffs while also reading
the adis16201 driver to learn how they implemented the connection inside the linux kernel.
datasheets:
https://www.analog.com/media/en/technical-documentation/data-sheets/ADIS16201.pdf
https://www.analog.com/media/en/technical-documentation/data-sheets/ADIS16203.pdf

with all of that being in the way I have a plan:
- compare datasheets.
- learn from both drivers adis16203 in staging and adis16201.
- use of https://www.kernel.org/doc/html/v6.9/driver-api/iio/core.html
- after I lose the fog I have because of my lack of understanding of the API I'll do a RFC
- Adjust the plan accordinly.

# 1- compare adis16201 and adis16203 datasheets:
  They are too similar in a lot of stuff so I'll just note
  things that will effect the code.
## similarities:
- Both uses SPI protocol
- The SPI interface provides access to:
  - temperature
  - power supply
  - one auxiliary analog input
- 
## differences:
- ADIS16201: access to measurements for dual-axis linear acceleration, dual-
axis linear inclination angle via SPI.
- ADIS16203: 360° linear inclination angles, ±180° linear incline angles via SPI.

# 2- learn from adis16201 and adis16203 drivers:

# 3- RFC:

# 4- Wait and Adjust:


