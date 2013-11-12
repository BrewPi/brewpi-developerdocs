Welcome to the BrewPi developer documentation!
================================================
This documentation is written in reStructured text format and version controlled on `GitHub <https://github.com/BrewPi/brewpi-developerdocs>`_.
To contribute to the documentation, fork the repo, make your changes and send a pull request.
You can also open an issue on GitHub if you don't have the knowledge or time to fix it yourself, but do want to report a bug.

The create UML diagrams in the documentation, we use `PlantUML <http://plantuml.sourceforge.net/index.html>`_.
You will have to install PlantUML on your system.
PlantUML also requires `Graphviz <http://plantuml.sourceforge.net/graphvizdot.html>`_ to run.

With PlantUML we can make pretty UML diagrams like this:

.. uml::

    class BasicTempSensor{
        +isConnected()
        +init()
        +read()
    }

    class TempSensor {
        +init()
        +isConnected()
        +update()

        +readFastFiltered()
        +readSlowFiltered()
        +readSlope()
        +detectPosPeak()
        +detectNegPeak()

        +{abstract} configMethods()

        -BasicTempSensor* _sensor;
        -TempSensorFilter fastFilter;
        -TempSensorFilter slowFilter;
        -TempSensorFilter slopeFilter;
        -unsigned char updateCounter;
        -fixed7_25 prevOutputForSlope;
    }

    TempSensor*--BasicTempSensor

Setup instructions for Linux:
-----------------------------
You can find a Debian package for PlantUML `here <http://yar.fruct.org/projects/plantuml-deb>`_. Install it with:

.. code-block:: bash

    sudo dpkg -i packagename.deb

Also install graphviz through apt-get. The python package sphinxcontrib-plantuml is easiest to install through pip:

.. code-block:: bash

    sudo apt-get install python-pip
    sudo pip install sphinxcontrib-plantuml


Setup instructions for windows:
-------------------------------
Install Graphviz with the installer.

Download the latest version of plantuml.jar and put it in the root of the repository.
This part of conf.py will set the PlantUML command for you.

.. code-block:: python

    if sys.platform.startswith('win'):
        plantuml = 'java -jar plantuml.jar'

Finally, install the python package sphinxcontrib-plantuml, for example with the python package manager in IDEA.

.. .. toctree::
    :maxdepth: 2
    :numbered: 2