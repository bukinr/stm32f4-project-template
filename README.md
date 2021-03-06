# STM32F4 Discovery board project template

[![Build Status](https://travis-ci.org/charleskorn/stm32f4-project-template.svg?branch=master)](https://travis-ci.org/charleskorn/stm32f4-project-template)

This is a project template for the [STM32F4 Discovery board](http://www.st.com/web/catalog/tools/FM116/SC959/SS1532/PF252419),
a development and evaluation board for a popular ARM microcontroller.

It should also be able to be adapted for use with any microcontroller from the STM32F4 series, although you may need to alter some configuration options.

It contains:

* a working toolchain for building and flashing software

* a working [Travis CI build](https://travis-ci.org/charleskorn/stm32f4-project-template)

* in `libs`, the [STM32F4 DSP and standard peripherals library](http://www2.st.com/content/st_com/en/products/embedded-software/mcus-embedded-software/stm32-embedded-software/stm32-standard-peripheral-libraries/stsw-stm32065.html),
  which has two main parts:

  * a set of header files with lots of useful constants (eg. registers defined by name)
  * a set of device drivers that abstract away some of the low-level hardware details

  The header files are quite useful, but I have mixed feelings about the device drivers. They do have some useful constants, and anything you don't use will be
  optimised out out of the final flash image.

* in `main`, a sample application that will flash the LEDs in a pattern to demonstrate everything is working OK:

  ![Flashing LEDs](doc/flashing-leds.gif)

* in `test`, a test runner with some dummy tests. This uses [Bandit](http://banditcpp.org/) as the test framework.
  It includes support for running the tests both on the host computer and on-device using [semihosting](http://www.wolinlabs.com/blog/stm32f4.semihosting.html).
  (See below for some important notes about this.)

This is a work in progress, but it should be ready for use. Please feel free to submit ideas, suggestions, issue reports and pull requests.

## Requirements

* [CMake 3.6 or later](http://cmake.org) - `brew install cmake` on OS X
* [GCC ARM toolchain](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm) - `brew cask install gcc-arm-embedded` on OS X
* [OpenOCD](http://openocd.org/) - `brew install openocd` on OS X

I haven't tested this on anything other than OS X. There's no reason I know of that would prevent it from working on Linux.
In theory, it should work on Windows as well, but we make use of Bash scripts in places, so you would either need to install
Bash (through [MinGW](http://www.mingw.org/), for example) or rework those parts to not use Bash scripts.

## Setup
You should only need to run this once to set up the build system:

```bash
./go.sh setup
```

## Building firmware
Run this command:

```bash
./go.sh build-target
```

## Flashing firmware
Connect the board to your computer with a USB-to-mini-USB cable (use the port at the top of the board, away from the buttons and audio jack -
the micro USB port at the front of the board is not for programming), then run:

```bash
./go.sh flash
```

## Testing on your computer (the 'host')
Run this command:

```bash
./go.sh test-host
```

## Testing on device (the 'target')
Connect the board to your computer just like you would for flashing firmware (see above), then run:

```bash
./go.sh test-target
```

Note that running the test firmware without a debugger that has semihosting support attached will cause the test
runner to hang. This scenario can be identified by the orange LED remaining on. Using `./go.sh test-target` should
take care of this for you.

[Bandit](http://banditcpp.org/) is quite large (takes around 190K of flash once all dependencies are included),
so you may want to consider switching to a more lightweight framework if this is an issue for your application.
It may also be possible to slim down Bandit by removing unused features.

In addition to printing information on the host computer, the test runner uses the LEDs to indicate the status of the
test run:

| LED    | Status |
| ------ | ------ |
| Orange | Enabling semihosting. Should only be on for a second or two at the beginning of the test run. If this LED remains on indefinitely, ensure that a debugger is connected to your device and semihosting has been enabled. |
| Blue   | Tests running. |
| Green  | Test run completed and all tests passed. |
| Red    | Test run completed but one or more tests failed, or no tests were found. |

## Tips / gotchas

### IRQ handler isn't being executed

* Make sure your handler matches the name given in `stm32f407vg.S`.

* If your handler is in a C++ file, make sure it is compiled with C linkage (see [this Wikipedia page](https://en.wikipedia.org/wiki/Compatibility_of_C_and_C%2B%2B#Linking_C_and_C.2B.2B_code) for an explanation of why this is necessary).

  This means you should wrap your IRQ handler in `extern "C"`, like this:

  ```cpp
  extern "C" {
    void MyReallyCool_IRQHandler() {

    }
  }
  ```

### Using with CLion
CLion uses CMake internally, and CMake's support for targeting multiple toolchains (eg. your computer and your target device) is
non-existent. Hence CLion doesn't support multiple toolchains either. (The `go.sh` script hides this detail from you, making
sure to always use a build tree appropriate for the task.)

You can choose to have CLion target either your local computer, or the target device.
I recommend targeting your local toolchain (especially if you're practicing TDD), but either is possible:

#### Using CLion with a local ('host') toolchain

No configuration is required to have CLion use your host toolchain - this is the default.

You'll be able to run and debug the tests using CLion's built-in tools. (Note that the tests won't appear in the 'Tests' tool
as that only supports Google Test.)

Until the fix in [CPP-2471](https://youtrack.jetbrains.com/issue/CPP-2471) is released, the test process output may appear
garbled, as CLion doesn't support the ANSI colour code escape sequences. If this annoys you, add `--no-color` to the list of arguments
in the run configuration of the test runner.

#### Using CLion with a device ('target') toolchain

##### Initial setup

Some tweaking is required to get [CLion](jetbrains.com/clion) up and running initially:

1. Open the project in CLion
2. In Preferences (OS X) / Settings (everything else), go to 'Build, Execution, Deployment', then 'CMake', and add the following
  to 'CMake Options' under 'Generation': `-DCMAKE_TOOLCHAIN_FILE=tools/toolchain-arm-none-eabi.cmake`

##### Debugging

On-device debugging is not supported in CLion (see issue [CPP-744](https://youtrack.jetbrains.com/issue/CPP-744)).

##### Flashing

Flashing from CLion is possible (run the `flash_firmware` task), but it will ask you for an executable to run.
You can either leave this blank, which will cause CLion to ask you for an executable the next time you attermpt to run
the task, or specify one of the executables produced (eg. `stm32f4test_firmware.elf`), although CLion will then try and
fail to run this executable on your development computer after flashing the firmware onto the board.

#### Issues with missing dependencies in CLion (applies to both host and target toolchains)

CLion creates the build tree in its own private directory and will not pull down dependencies specified with CMake's
`ExternalProject` unless you explicitly tell it to. This means that code completion will not work for libraries set up
using `ExternalProject` (such as Bandit) until you run a target that downloads that dependency - either the `<library>_sources` task,
or any target that depends on the library.

## Acknowledgements and references

* [ST's STM32F4 DSP and standard peripherals library](http://www2.st.com/content/st_com/en/products/embedded-software/mcus-embedded-software/stm32-embedded-software/stm32-standard-peripheral-libraries/stsw-stm32065.html)

* [ST's clock configuration tool](http://www2.st.com/content/st_com/en/products/development-tools/software-development-tools/stm32-software-development-tools/stm32-configurators-and-code-generators/stsw-stm32091.html)
  for generating `system_stm32f4xx.c`

* [https://github.com/adrienbailly/STM32-CMake-CodeSourcery/blob/master/CMakeLists.txt](https://github.com/adrienbailly/STM32-CMake-CodeSourcery/blob/master/CMakeLists.txt) for CMake snippets

* [http://tech.munts.com/MCU/Frameworks/ARM/stm32f4/](http://tech.munts.com/MCU/Frameworks/ARM/stm32f4/) for linker script and startup assembler

* [http://stackoverflow.com/a/20805828/1668119](http://stackoverflow.com/a/20805828/1668119) for compiler flags

* [http://simplemachines.it/doc/arm_inst.pdf](http://simplemachines.it/doc/arm_inst.pdf) for ARM instruction set (useful when tweaking startup assembly)

* [http://jeremyherbert.net/get/stm32f4_getting_started](http://jeremyherbert.net/get/stm32f4_getting_started) for pointers to some useful documentation and examples of how to use the GPIOs and timers

* [OpenOCD](http://openocd.org/) for original `stm32f4discovery.cfg` OpenOCD configuration file.

## Contributing

Any suggestions and pull requests welcome.
