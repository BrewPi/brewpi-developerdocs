A note on object identity throughout the systen
-----------------------------------------------

In this context, an object is a value object, such as a temperature or pressure reading.

id-chain
^^^^^^^^
In the embedded controller, objects are identified by the series of container indices that are needed to reach the
object, the so-called, id-chain. This form of ID is purely local to the embedded controller and the connector (python).

Local Object IDs
^^^^^^^^^^^^^^^^
In the connector, the id-chains are converted to *local object ID* to communicate with the service layer.

The service layer doesn't use the id_chain to identify an object but uses the local object ID that is
tied to the intrinsic properties of object itself. For a temp sensor, the ID might be the onewire address. For a digital actuator, the ID
might be the pin it is assigned to.

The id-chains are not used because they don't tell
anything about *which* device is connected - a device could be added at a certain id-chain location, then removed
and replaced with something else, which would then get the same ID if id-chains were used. Hence, the id is based on
the intrinsic properties of the object.

Some devices may use the id_chain if that is the only way objects can be distinguished
e.g. objects not related to any identifiable id, such as a Generic Container. However, it's anticipated that these
objects will not be needed by the service since they don't provide values.


Embedded Controller ID
^^^^^^^^^^^^^^^^^^^^^^
The next level of ID is for the embedded controller itself (since this is the logical container for all objects in it.)
The embedded controller's ID varies by controller type. For an arduino, it's a locally assigned unique ID (assigned
by the host). For the spark, it's the built in unique ID.

If the ID is only locally unique, then it is combined with the host's UUID to make a unique ID for the controller.

Host ID
^^^^^^^
If the ID of the embedded controller is only locally unique, it is combined with the host's
unique ID (e.g. MAC-address.)

For devices that don't have a host, such as a internet-connected device, the controller ID will already be unique
a UUID, and doesn't need combining with a unique host ID.


Unique ID for Every Device
^^^^^^^^^^^^^^^^^^^^^^^^^^
With this identification scheme of [host ID] : Controller ID : Local Object ID, a huge network of embedded controllers
with attached devices can be built up, each with a guaranteed unique identifier.

The Host ID, embedded controller ID and local object ID provide a hierarchy of IDs. This can be used to structure
the storage of logged data. For example, create a db schema for the host ID, containing tables based on each controller
ID, with fields based on the local object ID.

Local ID Mapping
^^^^^^^^^^^^^^^^
The connector is responsible for interring the local ID from a given object in a controller, inspecting it's configured
values to build the local ID. The connector maintains a mapping from id_chain to local ID.

The connector also maintains a mapping between controller IDs and the connection to the controller, such as:

    "Controller 2" -> COM4


Functional IDs
--------------
The IDs presented so far allow all values provided by all controllers to be uniquely addressed. The values can be
stored in a time series, fetched and plotted.

A parallel set of IDs are Functional IDs. These describe identifiable values in the Processes that the system manages.
A functional ID is a Process ID combined with a Process Value ID. For example "Fermenting Beer 1" : "Temperature Setpoint".

Functional IDs are like placeholders, or symbolic names for Device IDs. When a process is set up, all the functional
IDs exposed by that process assigned with a mappin to a full Device ID.

E.g. "Fermenting Beer 1" : "Temperature value" -> "Host 09-12-EE-F9-DD" : "Controller 1" : "Onewire 28ABCDEF00"


