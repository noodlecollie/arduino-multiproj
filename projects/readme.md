This directory contains projects. Each project should live in its own subdirectory, and should be comprised
of a project configuration file named `myproject_config.json`, along with any project-specific code.
The code may be structured in any way, and should still be found by the Arduino CLI at compile time.

Note that each project is expected to have the following functions implemented:

``` C++
namespace Project_myproject
{
	void setup();
	void loop();
}
```

These are equivalent to the `setup()` and `loop()` functions that are expected in the Arduino .ino file.
The .ino file is auto-generated in order to help keep the include paths consistent between code source files.
See the main readme file for more information.
