- I sent an RFC to make sure I read the datasheet
correctly and found a logical bug in the ad5504 driver, also because
I know the device bindings can be a mess I asked for guidence in it.
Because of cause if you find a problem you have to fix it. It's your duty
now.

- On the same day I've got a reply from David Lechner validating my
findings and drafting me the device binding needed. I just took what he
gave me, learned about it and went to start the work.

- Sent a v1 where I updated the device bindings with what
David advised. And added that into the driver.

- That was a mistake hhhhh, Krzysztof Kozlowski (The DT maintainer) did
redirect me to learn more about:
	- see dt-schema
	- see dac schema
	- vendor property vs standard property
	- how to use all of to get the same result with less code.

- adding the feedback from Andy that was mainly coding style:
	- Using macros like MILLI instead of hardcoding raw values.
	- using local pointers to achieve the 80 characters per line.
	- the IWYU principle


=> Compiling this will give the current tolearn list:
	| | Read dt-schema.
		|X| read writing-schema.rst
	|x| Read dac-schema.
	|X| Learning about vendor property vs standard property.
		=> in writing-schema.rst "The exact schema syntax depends on
		whether properties are known, common properties
		(e.g. 'interrupts') or are binding/vendor-specific properties."
		=> in the property schema section they discuss more about vendor
		vs standard property, and how to work with them.
	|X| Learn more about allof and integrate it to the dt-binding.
	|X| Learning IWYU principle

==> While reading the docs:
	- I need to add links to the datasheets in the description of the dt
binding for the ad5504 and ad5501.
	- need to read Documentation/devicetree/bindings/dts-coding-style.rst
		- later :)
	- For testing schemas:
	```
	make dt_binding_check
	make dtbs_check
	# we can run both at one :)
	make dt_binding_check dtbs_check
	# example:
	make dt_binding_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/iio/dac/adi,ad5504.yaml
	```


Based on the mailing list feedback from **Andy Shevchenko**, **Krzysztof Kozlowski**, and **Nuno Sá**, here is your comprehensive To-Do list for the next iteration of your AD5504 series.

todo:
* [x] Drop incorrect Suggested-by tags.
* [x] Keep Reviewed-by tag.
* [x] Revert vcc-supply to optional.
* [x] Rename clear-gpios to clr-gpios.
* [x] Add context to commit messages.
* [x] Verify usage of linux/device.h
* [x] Drop linux/linux.h if not used.
* [x] Add missing headers.
* [x] Double-check the alphabetical order of headers
* [x] remove pdata 
* [x] Introduce the struct device pointer in it's own cleanup patch.
* [x] Restore -ENODEV check.
* [x] use ARRAY_SIZE()
* [x] Strict validation: ensuring both range[0] and range[1] are correct.
* [x] Satisfy "The Purpose"; Simply fetching the GPIOs isn't enough => just removed this;
* [x] Preserve "Reversed Xmas Tree".
* [x] Compile Test: make W=1 C=1 drivers/iio/dac/ad5504.ko
* [x] Binding check: make dtbs_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/iio/dac/adi,ad5504.yaml
* [x] Hardware Test: Deploy in my rpi 5 and verify the scale math working
	  via sysfs.

// well forgot to update many things but:
finished that v3; had some problems:

1- I had to remove pdata but wanted to give the replacement in the same patch so the driver keeps working
correctly in all patches; but Jonathan told me to separate the patches;
2- I did introduce ABI breaking by remove the read_volage; I should keep it so old drivers can have it;
3- Sashiko bot did report a data race; found file ad5360 as an example of how to handle that; but how can I test this;
4- David showed me a new way of handling ACPI devices; and give me file: ti-ads7950.c to learn how it's used;

