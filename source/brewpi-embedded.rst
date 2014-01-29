BrewPi Embedded Devices
=======================
First draft proposal by Matthew McGowan

Here's a sketch of the next major revision of the arduino, aka brewpi-bot.

This is not intended to be a full spec, but a indication of thoughts and direction for further discussion.

Some key goals of the controller:

- flexible configuration of controller entities - sensors, actuators, controllers, etc..
- flexible configuration of non-controller elements - input/output devices such as encoders, displays etc.
- dynamic discovery of connected devices (I2C/Onewire)


Entities
--------
An entity defines a building block in the controller - maybe we call it Controller Object, or simply object.  Examples
are

 * Hardware Sensors (Temp, switch)
 * hardware actuators (digital pin on/off, digital pwm (fast), analog pin, onewire),
 * functional actuators (predictive, pwm (slow), heat-cool-composite),
 * functional temp sensors (filtering), controllers
 * controllers (pid, on/off, thermo-model, etc..)

This list isn't exhaustive - just to illustrate that there are many types of objects.

A key goal of the SPI is to allow these objects to be instantiated, and configured, and for each object to be assigned
an Id that is used to further identify it externally.

Some objects are themselves values - e.g. temp sensors and actuators expose a default value that is the temperature or
actuator setting. Objects may also contain values, for example, a temp sensor may expose values for adjusting calibration.
More complex examples are many values reported by a controller or used for configuration of the controller.

The external IDs are assigned by the arduino as each object is instantiated. These are top-level IDs, 1, 2, 3.. etc..
In essence, there is a hidden top-level container to which objects are added to by default.
If the object added it also itself a container, then it exposes it's contained entities via sub-ids, e.g. 2.1 references the first contained entity in object 2.

At a high-level the protocol allows the client to specify which objects to instantiate, and returns the ID - in pseudo-code::

    ts_id = create(type=temp_sensor, args=...)

Which objects provide sub-objects is part of the documentation for that object type.

Wiring Objects
--------------

To wire objects together to create relationships is done by passing the related object's id as a parameter, e.g.

    pid_id= create(type=pid_controller, temp_sensor=ts_id, etc....)

Note that I write this conceptually as a function call. In the serial comms protocol, it will be a separate send and
asynchronous response. The response includes the assigned external ID for the created object, plus enough info from the
original command so the caller can identify which request it belongs to.

Persistence
-----------
As each object is created, the same command used to create it is stored in eeprom. This allows all objects to be re-instantiated
when the controller powers up.


Values
------
With a controller, values are a key element in the problem domain. Values represent set points, process values,
manipulated values, configuration etc..

Values may be readable/writable or both. Values are abstracted to provide independence from how the value is computed,
and how the system reacts when that value is assigned.  Values are optionally readable,
and writable. They're readable/writable state is dynamic and may change at runtime.  Values may change readable/writable
state according to the state of other objects in the system. For example, currently, fridge SV changes from a writable value
in fridge constant mode, to a read-only value in beer constant mode. Similarly, beer_sv changes from readable/writable
in beer_constnat mode, to not readable nor writable (unused) in fridge constant mode.


Update Cycle
------------
To preserve consistency values are refreshed/computed once per update cycle. Once a value has been first computed/updated
during a given update cycle, subsequent requests to read the value during the same
update cycle will return the same value.

As an implementation detail, this is done using flags to indicate if a value has already been computed for this update
cycle. Alternative schemes using timeouts are too fragile to be used here, since
they will then be tied to knowledge of how long the update cycle is. Delays in the update cycle will cause the value to
be recomputed, providing an inconsistent set of results. Ensuring all values are stable for the duration of the update
cycle is a key tenet to ensuring consistency and repeatability.


Display
-------
To avoid a hard-coded menu system, the display text is written as a template, which includes placeholders for values.

For example, the current display could be written as::

    Mode   %15r_mode_mdstr
    Beer   %4r_beer_pv   %4r_beer_sv  %_tempunit
    Fridge %4r_fridge_pv %4r_fridge_sv %_tempunit
    %15l_state_statestr %7r_statetime_hms

The conceptual format for a place holder is::

    '%' - (distinguishes start of placeholder)
    width  - number of chars to output on the lcd
    alignment - l,r for left, right
    value - an identifier for a readable value in the system
    format - an identifier for a formatter to use to format the value

Note that this is pseudo-code - the internal representation would most likely be more compact, e.g. using the MSB of a byte as an excape code
or some similar scheme to efficiently encode the placeholders.

The display checks values for changes and updates only those parts that have changed.

With the use of a template (stored either in code, or fetched from eeprom) allows flexibility to configure the display according
to the runtime application. This might be as trivial as changing "beer" to "wine" for a 2 stage controller in a winery, or reworking
the display completely for a different application.

Rotary Encoder/Menu
-------------------
The menu system can use the same display template to drive the menu.

When the user hits the rotary encoder, the first template value that is a WritableValue (indicating that the value can be assigned)
is found.  The portion of the display is then flashed (mode setting in the example above). Rotating the encoder
calls increment/decrement on the associated value object to fetch next/previous values, and these are then formatted as per
the display template.  As with the current menu code, a timeout is used to restore the value to the original.

The key message here is that the menu can be driven also from the template, generically, without requiring the display to be hard-coded.

After setting mode, pressing the encoder again jumps to the next writable value. For example after setting the mode to "beer constant",
the controllers mark fridge_sv as non-writable, so the next writable value is beer_sv. After setting this value, pushing the encoder again
jumps to the next value, but as there are no more writable values, the menu exits.


Serial Comms Format
-------------------
The serial comms format will be a mixture of text and hex-encoded binary.

The hex-encoded binary avoids parsing and allows the same data to be stored in memory, transmitted over serial, and
persisted to eeprom. To assist with testing, the format will have quoted text as comments to allow text strings to be
inserted free-form into the format, which are ignored by the controller.

For example: to define a temp sensor, the format looks like::

    <add>0A<id>00<OneWireTempSensor>01<address>28C80E9A0300009C\n

Here `0A` is the (arbitrary) value for the add command. The syntax for the command expects the subsequent byte to denote
which slot (the id) the object is stored in. When it's 00, the id is assigned by the arduino. When it's non-zero the
device at the existing slot is replaced with the new one.

The next byte defines the type of device to create. Each object type (class) has a unique type ID which is given here.
In this case, 01 denotes a Onewire temp sensor.  The remainder of the line are parameters specific to the type of the
device being configured. Here, the OneWire temp sensor takes an additional argument, the onewire address.

On receiving the command, the controller does any validation needed, and instantiates the requested object. Once instantiated,
the response is sent::

    <added>0A<code>00<id>22

This indicates that that the add command returned a zero (no-error) status code and the id of the created temp sensor
is 0x22.

Other protocol details
----------------------
The newline after each complete command is mandatory and serves to separate commands. This allows variable length commands to be supported.
The alternative is to include a byte count as part of the initial command definition - I feel the terminator character is more straightforward and
enhances readability.

To ensure each response can be easily paired with each request, each request includes a unique command ID that is defined
by the client. When the arduino sends a response, it includes the unique command ID provided by the client.

The arduino may also output <annotated> hex bytes in DEBUG mode, or in production builds, if space permits.
If there is insufficient space in the arduino for the annotated protocol, then it is annotated by the python arduino
handler. The text in angled brackets is optional and is there simply to make reading the format simpler. It's optionally generated
by clients. (And optionally verified by the arduino in DEBUG mode.)

Composite IDs
-------------
Global IDs are simple values, e.g. ``0x10``. To allow IDs to also reference objects contained within a container we need
composite IDs. E.g. 0x1001 refers to the object at location 0x01 in object (container) 0x10.

For the serial protocol, these composite IDs need to be encoded. Because the IDs are no longer fixed length, the parser
needs to know how long they are. A simple scheme is to set the MSB to 1 if the ID has another byte following. So 0x1001
would be encoded as 0x9001.  (An Alternative scheme would be to prefix each ID with the number of bytes, essentially making them
array parameters - this is also workable.)

Serial Comms and Generic Function Call Mechanism
------------------------------------------------
In essence, the serial comms is a generic function call mechanism, using the same encode/decode logic for all functions
offered by the arduino. This will dramatically reduce the size of encode-decode logic used, so there is more room for
useful functionality.

There is a global function table that defines the function's external ID (e.g 0x0A for Add Object) and the function argument
types and return value. This is used to (optionally) verify the arguments passed in, and guides the decode process. Recall
that each object type that can be externally instantiated has an associated typeID so these can also be part of function
call arguments and results - all entities are referenced by their ID.

Some examples of the functions supported:

 - add object - constructs a new object and returns its external ID (or returns an error code if this is not possible.)
 - read values - reads a number of values (each referenced via their ID) and returns each value. (the result includes status
    flags for invalid IDs and values that cannot be read.)
 - write values - write a number of values in one hit
 - reset values?? (I would prefer the default values being exposed so a  reset is simply writing the defaults.)
 - read default value - fetches the default value for a given value object.

The read values/write values replace much of the existing functionality
 - t - list SV/PV temperatures and state - read values from temp sensors (either the base hardware sensor, or the filtered sensor, both
    are available). State is fetched by reading the values of actuators.
 - j - update settings - write values
 - l - read display - now read "buffer" value on a display object.
 - A/a - enable/disable alarm - write value to alarm actuator
 - C/S - reset constants/settings - write values to defaults (either defaults stored in script, or fetched from arduino)
 - c/s - read constants/settings - read values
 - n   - print version string - we may keep this as is, or create a special value for the version.


 Commands replaced by top-level functions
  - E   - initalize eeprom.
  - h{} - list detected hardware. This can either be a new top-level function, or a special value container
    can be instantiated (by the script) whose job is to provide details of the hardware detected.
  - d{} - list configured devices (from eeprom.) This is a new top-level function that outputs the device recipe stored in eeprom.
  -  U{} - define a device - replaced by the general Add object function.


Templates/Recipes
-----------------
Rather than having to repeat commands to define many objects to build (say) a 2-stage temp controller over and over for each
instance of the controller required (e.g. each fridge), it would be nice to create recipes containing many objects.
These define input and outputs to the whole recipe and hide the internal objects used and complexity of the wiring.
This reuse may save eeprom space.

It's not clear yet if these recipes should exist on the arduino or externally.
