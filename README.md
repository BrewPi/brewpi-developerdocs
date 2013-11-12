brewpi-userdocs
===============

This repository contains the BrewPi user documentation in reStructuredText format.
It is compiled to HTML by Sphinx, a python documentation generator.

A bat file is included to generate the documentation on windows.
You will have to install the python module sphinx, which for example can be done using easy_install.

After installing sphinx, set the environment variable SPHINXBUILD to the full path of sphinx-build.exe, e.g. :
C:\Python27\Scripts\sphinx-build.exe
A reboot might be necessary.

You can compile the HTML documentation with:
make.bat html

Contributions to the documentation are very welcome! Please send me pull requests on GitHub!

This repository is for end user documentation. Our build server builds docs.brewpi.com from this repository.
It also contains the changelog for the project, which is updated each official release.

Developer documentation (API, function descriptions, etc) will be added to the code repositories, not here.

IntelliJ IDEA has a plugin by JetBrains for reStructured text, which you can install through the plugin manager.
BrewPi has an open source license for IDEA. if you want to use it for BrewPi, you can request the key by e-mail.

