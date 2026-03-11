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
* [ ] Drop incorrect Suggested-by tags.
* [ ] Keep Reviewed-by tag.
* [ ] Revert vcc-supply to optional.
* [ ] Rename clear-gpios to clr-gpios.
* [ ] Add context to commit messages.
* [ ] Verify usage of linux/device.h
* [ ] Drop linux/linux.h if not used.
* [ ] Add missing headers.
* [ ] Double-check the alphabetical order of headers
* [ ] remove pdata 
* [ ] Introduce the struct device pointer in it's own cleanup patch.
* [ ] Restore -ENODEV check.
* [ ] use ARRAY_SIZE()
* [ ] Strict validation: ensuring both range[0] and range[1] are correct.
* [ ] Satisfy "The Purpose"; Simply fetching the GPIOs isn't enough
* [ ] Preserve "Reversed Xmas Tree".
* [ ] Compile Test: make W=1 C=1 drivers/iio/dac/ad5504.ko
* [ ] Binding check: make dtbs_check DT_SCHEMA_FILES=Documentation/devicetree/bindings/iio/dac/adi,ad5504.yaml
* [ ] Hardware Test: Deploy in my rpi 5 and verify the scale math working
	  via sysfs.


