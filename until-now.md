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

# adxl345 v3 2026-02-14

- Finally got a reply from Jonathan with a "Nice series thanks!",
and applied the series to the testing branch of iio.git.

- However, almost simultaneously, David Lechner, Andy Shevchenko,
and Randy Dunlap posted reviews. They didn't find logic errors, but they
found important stylistic nits:
  1. David noted that `BIT(IIO_EV_INFO_SCALE)` should be ordered
  consistently after `VALUE` in the driver and that I missed documenting
  `in_accel_mag_adaptive_scale`.
  2. Andy and Randy caught a subtle whitespace error where I accidentally
  replaced a space with a tab on a line I shouldn't have touched.

# adxl345 v3 2026-02-15

- The "race condition" resolved. Jonathan Cameron saw the incoming
feedback after applying the patches and wrote: "Ah. Raced with other
feedback. Dropped again for now."

- This means the code was removed from the testing branch to allow me to
address the reviewers' comments. I took it as an opportunity to make a
cleaner version.

- I replied to the thread confirming I understood the situation and
thanked the reviewers for their detailed feedback. I am now preparing V4
to fix the bitmask ordering, the whitespace consistency, and the
documentation.

- Well even after checking the patches over and over I just didn't see
those mistakes. It's great to learn from them.

- Jonathan (using static analysis) discovered a functional bug that
everyone else missed. I had enabled the `SCALE` bitmask for Activity
(MAG) events, which created the sysfs files, but I failed to update the
`read_event_value` function to actually handle that case. As a result,
reading those files returns `-EINVAL`.

- Jonathan advised me to "up my testing game," highlighting that I
shouldn't have advertised a feature I hadn't verified in the code path.
Which you know hurts, hurts cause I know I can do better :).

- This day 02-15 was a big hit to me, and a reality check you could say,
I'm going to take stuff slower right now. No need to rush patch series.
I'll start spacing the series by at least 48h after the last reply I get,
that time will be well spent learning more about the subsystem and it's
history. I was going too fast and juggling too much, seriously from style
rules to ABI, to DT schemas and core logic... Total overload, but I did
learn many lessons in the way. The plan now it to not stop, but to slow
down.

# ad5504 v1 2026-02-15:
- Received feedback from Jonathan Cameron regarding mailing list
etiquette. He advised against sending separate "Thank you" emails to
reviewers, as it increases the already high volume of mail for
maintainers. Instead, gratitude should be expressed in the patch changelog
(below the `---` line) or in the cover letter of the next version.

- Jonathan also weighed in on the scale calculation logic.
He suggested explicitly checking for valid values (30V vs 60V) rather than
assuming one if the other isn't present, to protect against invalid DT
properties.

- Interestingly, my current local implementation already aligns with this:
I defaulted to 60V and only switched to 30V if the property explicitly
matched the 30V range. It's a good confidence boost to see my logic
matching the maintainer's expectations before even sending the V2.

# adxl345 v4 2026-02-21:
- I took 2 days rest, after that I started working on a dummy-iio driver
where I did rewrite all the needed parts of the adxl345 into it, so I can
check the actual output of my work, there I found that the names generated
didn't match what me or the maintainers expected to be.

- Used the new sysfs for the table and updated the docs to match the real
output. I also did work on fixing the bug stated by Jonathan, and explicitly
rejected writing to those files. And of course had to re-order the
bitmasks.

- Today I did send my v4 :)

# adxl345 v4 2026-02-21:

- David reply asking about the double tap scale sysfs I did make, that had
no double tap value, that could be referring to it.

- I did go over the driver and the datasheet one more time, and found that
the actual value of the double tap, was already the same at the one you'll
find in the single_tap sysfs. So I concluded that the driver didn't expose it
to remove dublicated files that outputs the same thing. I was going to
remove the scale I added. To keep everything as it was.

- Jonathan did reply specifying that "The IIO ABI always allows any write to any
channel property including event thresholds to change the value of a different
one, so there is never a problem with duplication like this". So The plan now
is to add a new sysfs file that exposes the double tap value as well.

# adxl345 v5 2026-02-24:

- I sent a v5 that added the double tap value, and added documentation for it
as well. Also re-organized the patch series. And the made it a 5 patch series
instead of 4.

- Only Andy that did reply giving me some advice about making the patch cleaner
and touch only what's necessary. As that makes the patch easier to review.

- Also stated that the mentorship CC was debouncing, and advised me to remove it.

- I did fix everything then reply to him just before sending a v6, where I
explained about the mentorship and promised a v6 without the CC.

# adxl345 v6 2026-02-26:

- Did clean up my patches as Andy advised, and sent a V6.

# adxl345 v6 2026-02-28:

- Jonathan Cameron did reply stating that he added the patch into the testing
branch, and after a few days of bots poking at it. He will push it out as
togreg which is the branch linux-next picks up.

- After Jonathan message by a few minutes David replied to me with a reviewed-by.

# adxl345 v6 2026-03-01:

- Jonathan updated to add David's tag on my work.

# LFX mentorship 2026-03-01:

- I received an email from Shuah Khan congratulating me that I've been one of the
selected mentees for the spring mentorship starting from March 3rd to the 1st of
September.

- She stated that I have to chouse two areas of the linux kernel to work troward
getting 5-10 patches in for the next 6 months. I'm taking the IIO subsystem for sure
especially that my ad5504 driver work is looking now like a 5 patch series alone, For
the other area I'm thinking of taking kselftest for many reasons, not only writing test
for the kernel will make me a better kernel developer, it's also the subsystem Shuah
maintains. I'll have a meeting with the actual maintainer each week :)
