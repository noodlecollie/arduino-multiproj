Arduino-Multiproj
=================

Arduino-Multiproj is a small set of scripts intended to assist in building Arduino projects
that are comprised of multiple smaller, independent sub-projects.

Although the Arduino tools are good at allowing users to get up and running quickly with
minimal effort, the compiler is not very flexible when it comes to building in multiple
configurations, or sharing code between multiple projects. These build scripts were written
in order to allow this to happen: depending on the project selected, the relevant code
files are selected and copied into the correct directory structure in an intermediate build
directory, and the compiler is run on this directory tree.

## Requirements

Python 3 is required to run these scripts. The first time the scripts are run, an internet
connection is required in order to download the appropriate `arduino-cli` executable from
the [arduino-cli project page](https://github.com/arduino/arduino-cli).

## Structuring Projects

A directory tree for a multi-project workspace might look like the following:

```
buildsystem
    ... # Arduino-Multiproj scripts are here.
modules
    SharedModule1
        SharedModule1.h
        SharedModule1.cpp
projects
    myProject
        myProject_config.json
        src
            main.cpp
    sanityTests
        sanityTests_config.json
        src
            main.cpp
build.py
```

An explanation of the directory structure is as follows:

* `modules`: Units of code that may be accessed by multiple projects. Each project specifies
  which modules it makes use of in its configuration file (see later).
* `projects`: Each subdirectory corresponds to an individual project. Each project has a
  configuration file named `projectname_config.json`, which stores information about how to
  compile the project. Per-project code files are stored under the `src` subdirectory.

To compile `myProject` and upload to a connected device, `build.py` would be called as follows:

```
python build.py --project myProject build upload --port COM3
```

Upon compilation of `myProject`, the relevant files are copied to the build directory ready
to be compiled. This directory structure allows the `arduino-cli` to correctly find and compile
all of the source files.

```
build
    myProject
        myProject.ino # Generated automatically - see later.
        src
            main.cpp
            SharedModule1
                SharedModule1.h
                SharedModule1.cpp
```

## Commands

Anything provided to `build.py` as an positional argument (ie. without a preceding `-` or `--`)
is interpreted as a command. Multiple commands can be specified: for example, `build upload`
would build the specified project and then upload it afterwards.

Available commands are:

* `clean`: Cleans the build directory of compiled files.
* `build`: Copies relevant files to the build directory and runs `arduino-cli`, downloading this
  first if necessary. Only files that have been modified in the source tree since the last build
  are copied to the build directory.
    * If the specified project is different to the project that was built last, the build
    directory is automatically cleaned.
* `upload`: Uploads to a device connected to the computer on the port specified by `--port`.

Available options are:
* `-p/--project`: Always required. Specifies which project to run the specified commands for.
* `--port`: Only required when calling `upload`. Specifies the serial port on which the target
  device is connected, eg. `COM3`.

### A Note On .ino Files

The project's `.ino` file is generated automatically by the build scripts. This is in order
to help keep include paths consistent between all files used in a project, because at the
time of writing I cannot find a nice way of providing extra include paths to the `arduino-cli`
executable without overwriting the existing compiler flags that it uses.

As an example, if the `myProject.ino` above were written by hand, the include path for
`SharedModule1.h` would need to be:

``` C++
#include "src/SharedModule1/SharedModule1.h"
```

This is not obvious without either looking at the build scripts themselves or the output
directory structure they create. Therefore, the main `setup()` and `loop()` functions have
been moved from the project's `.ino` file, and should instead be implemented somewhere
within the project's code as namespaced functions.

For example, in the case of `myProject` above, `main.cpp` would function as the equivalent
of `myProject.ino`. The functionality is identical, except that the `setup()` and `loop()`
functions should be declared within the namespace `Project_myproject`:

``` C++
// Including the shared module is now straightforward:
#include "SharedModule1/SharedModule1.h"

namespace Project_myProject
{
    void setup()
    {
        // Setup code goes here.
    }

    void loop()
    {
        // Loop code goes here.
        SharedModule1::doSomeStuff();
    }
}
```

If no namespaced `setup()` and `loop()` functions can be found by the compiler, linking
will fail.

As a general rule, when including header files within a source file, specify the path
relative to the location of the source file. If `SharedModule1.h` wanted to include a
header for another module, for example, the path would be `../OtherModule/OtherModule.h"`.

## Project Configuration

Within each project directory, a configuration file should exist for that project. The
file should be named after the project: in the case of `myProject` above, this would be
`myProject_config.json`.

The configuration items should be stored within the file in a root JSON object. For
example:

``` json
{
    "modules":
    [
        "SharedModule1"
    ],

    "fqbn": "Heltec-esp32:esp32:wifi_kit_32",
    "warnings": "all"
}
```

Supported configuration keys are:

* `modules`: Required. A list of strings specifying the names of modules to compile when
  building this project.
* `fqbn`: Required. Fully-qualified board name for the target board.
* `warnings`: Optional. Warning level to use when compiling this project. Default is `all`. Possible
  values are `none`, `default`, `more` and `all`.

## Future Improvements

* Support more platforms than Windows. This would just require adding the relevant
  `arduino-cli` URLs for the specific platforms.
* Download the latest `arduino-cli` executable. Currently `0.9.0` is used.
* Add support for building projects in different configurations, eg. debug vs release.
* Add support for creating a project directory and sample configuration file via `build.py`; for
  example, `build.py --project newProject makeproj`.
* Refactor scripts so that `--project` is not required unless explicitly needed by the commands
  being executed. For example, the `upload` command currently requires the project to be specified
  in order to get the `fqbn` for that project; instead, the project config should be copied to the
  build directory when `build` is executed, and `upload` should look for this in a known place.
