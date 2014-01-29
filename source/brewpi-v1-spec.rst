BrewPi v1.0 Functional Specification
####################################
This document is a functional specification for the next version of BrewPi, v1.0.
This document is intended to be a starting point for the design, not a final blueprint.
We will keep the document up-to-date while we develop BrewPi v1.0.

-- Elco Jacobs

Working on this document
************************
This document is part of the BrewPi developer documentation, which is version controlled on GitHub.
Contributions to this document are managed like any other code, via forking, pull requests and GitHub issues.
This document is compiled to HTML and published on devdocs.brewpi.com by our Jenkins build server.

This gives us a few advantages over working with a Wiki:

* Use your favorite text editor with reStructuredText syntax highlighting (PyCharm for example).
* Use git/GitHub for collaboration. This gives us a place to manage contributions and to discuss issues.
* Easy to integrate UML with plantUML

Overview
********
BrewPi v0.x has evolved into a pretty awesome fermentation temperature controller, but the way it is designed does not scale.

The main shortcomings of BrewPi v0.x are:

* Lack of modularity and flexibility. A lot of the functionality in BrewPi v0.x is hardcoded.
  For example the temperatures that are logged and/or controlled are hardcoded: fridge, beer and room temperature.
  In BrewPi v1.0 the goal is to be able to freely add your sensors, name them and assign them to controllers.
  BrewPi v1.0 should be a swiss army knife for brewing: add the sensors and actuators you have in your setup, add controllers and choose which data you want to display in the web interface.
  The architecture should be able to control a fermentation fridge, a BIAB mash, a HERMS, a RIMS, a hot plate, etc.
* Lack of testing. We got started with unit testing with googletest and pyUnit, but most parts of the code are not tested by unit tests.
* Lack of scalability. BrewPi v1.0 should be able to control more than a single process.
  In a brewery multiple processes should be able to be controlled and monitored by BrewPi.

We decided to redesign the BrewPi architecture to address these issues.
This will be a rewrite from scratch, but of course a lot of the v0.x code can be reused.

.. epigraph::

    BrewPi v1.0 is a software framework to manage multiple process in home and micro breweries.
    BrewPi compatible hardware will control one or multiple processes standalone and will report to a service layer.
    The service layer logs the data it gathers from the controllers and provides access to the controllers via a RESTful API.
    On top of the service layer, multiple GUIs can display logged data and provide a user friendly way to manage processes.

Major Components
****************
BrewPi will be split into separate layers, that have well defined APIs in between.
All control logic will still be put in embedded devices, for maximum robustness.
There can be multiple of these embedded devices, that will all report to a service layer.

The service layer is the glue between all other components. Every other component will connect to the service layer.
There is only one service layer.
The service layer provides a RESTful API for other services to connect to.
The service layer logs data into a database.

.. uml::
    cloud "Hardware Controllers"{
        [Embedded device 1] as ed1
        [Embedded device 2] as ed2
    }

    cloud "Clients"{
        [Web Front End] as frontend
        [Android / iOS App] as app
        [Third party app] as thirdparty
    }

    package "BrewPi server" as server{
        package "BrewPi service" as sl{
            [Authorization]
            [Authentication]
            [BrewPi Manager] as manager
            () "RESTful JSON" as rest
            () "Device protocol" as deviceprotocol
            database "Time series db" as tdb
            database "Relational db" as rdb
            [USB Adapter] as usbadapter
            [WiFi Adapter] as wifiadapter
            manager -up- deviceprotocol
            manager -down- rest
            usbadapter ..  deviceprotocol
            wifiadapter .. deviceprotocol

            manager <-left-> tdb
            manager <-right-> rdb
            }
        package "Web server" as webserver
    }

    ed1 -- usbadapter : USB
    ed2 -- wifiadapter : WiFi

    rest ..left.. webserver
    webserver ..down.. frontend
    rest ..down.. app
    rest ..down.. thirdparty


Embedded devices
====================
The embedded device in BrewPi 0.x is the Arduino.
In BrewPi v1.0 other hardware options will be added, first candidate is the Spark Core.

The embedded device is independent for process control, it can run standalone.
Without being connected to other layers, it will be able to regulate temperatures.
Sensors, Actuators and controllers will therefore be stored on the embedded device.
Configuration of these sensors, actuators and controllers will be done via the API, but once they are defined, the device can run without connectivity.

The embedded device does not log data locally, other than what is needed for control.
An exception might be that bigger future devices can buffer data for sending it to the service layer in batches.

Service layer
=============
BrewPi will be installed as a system service on a central server.
This central hub is responsible for collecting data, monitoring processes and providing an interface for GUIs to connect to.


Web interface
=============
The web interface consumes API provided by the service layer and builds a GUI based on the data received.
An option is to store views in this layer. This layer will not store data.

The web interface will be built as a web app, with a JavaScript MV* framework.
It will consume the RESTful API provided by the service layer.
It will stay in sync with the service layer: the data is 'live'.

Other GUIs
==========
Phone and tablet apps can also consume the RESTful API of the service layer and build a custom interface for tablets and phones.

Database
========
Each process that runs on the embedded devices outputs data. The service layer stores this data in a database.
There will be a time series database for process data and a relational database for settings.

Major features
**************

Processes
=========
Core to the functionality of BrewPi will be processes.
Each process will have sensors and actuators assigned and will have controllers in between. It also has a list of settings.
Each of these can output data: measurements for sensors, output values for actuators, internal variables and outputs for controllers.
A process can be mashing, sparging, fermentation, etc.

Settings
--------
Settings are values that are changed in the web interface or via an LCD menu.
A setting is for example a beer temperature setting or a list of profile points.
They can only be changed by the user, not by the algorithm.

Internal settings of controllers (like Kp,Ki,Kd for a PID controller) are not considered a setting here.

Sensors
-------
Sensors measure things in the real world: temperature, bubbles, specific gravity, switches.
Sensors can be assigned as input to controllers, but they can just measure data for logging.

Actuators
---------
Actuators are devices that change things in the real world: heaters, coolers, lights, fans, solenoids.
There will be different types of actuators, like on/off or PWM.

Controllers
-----------
Controllers will be at the heart of BrewPi: they read inputs (settings, sensors or other controllers), perform an internal algorithm, and drive outputs.
These outputs can be read by actuators.
Controllers have internal settings for constants in the algorithm.
There will be different types of controllers: PID, model based, predictive ON/OFF.

Multiple controllers can be tied together to from more complex control algorithms.
For example, the output of a beer PID controller can be the input for a fridge temperature controller.

Filters
-------
In BrewPi, most temperatures are filtered by low pass filters.
These filters have adjustable filter frequencies, one input and one output.

Data output
-----------
Each of the entities described above will be able to output its values.
These values can be sent to the service layer when requested (pull) or when a significant change occurs (push).
The service layer can log the data into a database or provide a transparent interface via the REST service.

Data logging
============
The BrewPi service will log all changes that occur into a time series database: temperatures, settings, internal variables.
This data can be queried through the RESTful service in the service layer and rendered in a GUI.

The time series database has to be able to provide any subset of the data on request, so custom charts can be added in the GUI.

RESTful Interface
=================

Graphical User Interface
========================


Install and updates
===================


Design Goals
************

Modular and Customizable
=========================
BrewPi should set itself apart from other temperature controllers in that it can be adjusted to match your custom setup.
Brewing setups vary greatly in type of sensors, heaters, coolers, tanks, etc.
An example of the processes BrewPi should be able control:

* A HERMS (Heat Exchange Recirculating Mash system): 3+ temperature sensors, 3 tanks, 2 heaters, 1 or 2 pumps.
* A RIMS (Recirculating Infusion Mash system): 2+ temperature sensors, 2 tanks, 2 heaters, 1 pump.
* A BIAB (Brew In A Bag) system: 1 temperature sensor, 1 heater.
* A fermentation fridge or freezer: 1 chamber, one or more beers, 2+ temperature sensors, 1 cooler, 1+ heaters, fan, light.
* A glycol distribution system: 1+ sensors, 1 cooler, 1 pump, 1+ solenoids, 1+ tanks
* A kegerator or keezer: 1+ temperature sensors, level sensors, RFID tags

As you can see, these setups differ greatly in types and number of actuators, sensors, processes and sub-processes.
Each process will require a custom interface that shows all relevant data and settings.
On top of that, most brewers will have multiple systems (mashing and fermentation), which should all be accessible from the same interface.

The architecture should allow for such a variety of processes to be specified, controlled, logged and visualized.

Integration
===========
Each brew involves multiple steps, which should be aggregated in the web interface.
For one brew you should be able to easily navigate to the:

* Recipe
* Mashing, Sparging, Boiling, Cooling
* Fermentation
* Conditioning

A user should be able to view all data for his brew in a few clicks, so he can view all factors that might affect the taste at once.
To achieve this integration, we can collaborate with others. For example, we could connect with BrewToad or BeerSmith.


Ease of use
===========
Despite being flexible and highly customizable, BrewPi should not be much harder to use than other temperature controllers.

Defining control a process as a combination of modular building blocks will be a complex task, so we should provide a tool to help users set up.
This can be in form of a wizard and by providing example configurations.
For example: the user selects the process fermentation chamber and can then choose/assign sensors and actuators in the GUI.
He selects that he has one carboy in an 100W fridge, with a FermWrap around the carboy as heater.
Next he assigns pins to actuators and selects which temperature sensor corresponds to the beer and the fridge air temperature.
In the web interface a new view is added for this process, with a default view already loaded.
The user can later customize the modules in the view if he likes.

A visual representation of the process with clickable components will probably provide the most user friendly interface.

Reliability
===========
Tests, Simulation, Independent components, Alarms, Damage control

Extensibility
=============
The architecture of BrewPi should allow users to write custom modules for BrewPi.
These can be new types of hardware, new GUI components, new data export tools, etc.
This design goal is related to the modularity goal.

Scalability
===========
BrewPi targets home, nano and micro breweries. It should be able to scale up to managing dozens of processes, logging hundreds of temperatures.
A long term goal is connecting to an optional cloud service, which will log data for thousands of breweries.

Users
*****

Home brewers
============

Micro breweries
===============

Licences
********
GPLv3, arguments


Physical Architecture
*********************
REST + Service layer: Raspberry Pi, Other servers

Embedded devices: Arduino, Spark core

Clients: web server, phone app


Software Architecture
*********************


Coding conventions
******************


The User Experience
*******************


Future features
***************
Recipe integration (collaboration for example BrewToad)
Other processes: charcutery, cheese, sous vide cooking, greenhouses, etc.

