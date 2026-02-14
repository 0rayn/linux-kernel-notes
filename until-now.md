# adxl345 V1 2026-01-27:
- While reading the documentation of the adxl345 I found a problem in the
event scale between the documentation and the driver's datasheet. So I did
fix it with the grammar issues and typos and sent a patch.

- This was just scratching the surface, as Jonathan Cameron (the IIO
subsystem maintainer) did point out that this revealed deeper issues
regarding not only the documentation but the driver itself. After further
investigation I tried to implement the fix for the ABI compliance of the
driver, but instead of just creating a sysfs interface manually to expose
the scaling I opted to add support in the IIO core for such an attribute.

# adxl345 V2 2026-02-01:
- After making my changes, updating the math, fixing the driver, adding
the support to the IIO core and refixing the docs I did send a V2.

- Here it was my first debate with David Lechner about whether adding the
support in the IIO core was the right move or was just something that
no one will use except me. I did my research and backed-up my
proposal with another driver I found that suffers from the same need.
The mma8452 also was forced into static, manual workarounds to comply with
ABI.

- David accepted my reasoning, but waited for Jonathan to weigh in on this.
Jonathan did reply and say that it makes sense to add this to the IIO core.

# adxl345 V3 2026-02-08:
- I did fix typos, and re-structured the docs as the reviewers did advice
and sent a V3. Waiting for the review.

# ad5504 RFC 2026-02-10:
- While reading the code and the datasheet of the ad5504 I believe I found
a logical problem regarding how the driver computes the output scaling.

- The datasheet states that the ad5504 has an integrated precision
reference, and what determines the output ranges is the state of the R_SEL
pin (0-30V or 0-60V); while the driver calculates the scaling depending on
the vcc regulator voltage.

- If my understanding is correct, using vcc for scaling is flawed.
For example, a 40V supply currently results in a ~9.7mV scale
(40V / 4096), whereas the hardware output (in 30V mode) is fixed
at ~7.32mV (30V / 4096).

- So I sent an RFC to make sure I understood the problem correctly and
want to take feedback on how to determine the R_SEL state.

- David Lechner replied quickly confirming that I read the datasheet correctly.
He pointed out that this pin could be hard-wired or controlled by a GPIO.
He suggested using `output-range-gpios` and `output-range-volts` in the
Device Tree bindings (using `allOf` to make them exclusive) to handle this.
He also pointed out other missing bindings I should fix while I am at it
(vlogic-supply, clear-gpios).

# ad5504 v1 2026-02-12

- finished the v1 with bindings changes and driver rework and sent it.

- The feedback from Andy Shevchenko was almost instantaneous, while the
logic was correct, Andy gave me a masterclass in kernel coding style.
He pointed out that the headers were a mess and wanted me to add a patch
or two to re-structure them and suggested the use of MILLI macro instead
of hardcoding raw values. And showed me a cleaner way to handle property
reads using a local struct device *dev pointer. This last one is something
we used to do in 1337 to stay under the 80 character per line but I wasn't
sure of using in the kernel but this is great.

# ad5504 v1 2026-02-13

- On the bindings side, Krzysztof Kozlowski (the DT maintainer) was
equally fast and strict. He pointed out that I didn't need custom vendor
properties because standard ones already existed in the dac.yaml schema.
(Well I didn't even know about these standard schema's existance so this 
is great to learn). He also caught my mistake of using a vendor prefix on
GPIO properties(apparently, standard GPIOs like range-sel-gpios shouldn't
have them. <TODO> I have to learn more about what 'standard' and what not)
He even gave me a tip on using a more concise not: required: logic for
mutual exclusivity.

- I acknowledged their feedback and confirmed I’ll be restructuring the
series for V2. I plan to split it into a 4-patch or 5-patch series to
include dedicated cleanup commits for the headers and the
"Include What You Use" (IWYU) principles Andy mentioned. It’s a lot of
rework, but seeing how fast these senior maintainers reply when the logic
is sound is incredibly encouraging.
