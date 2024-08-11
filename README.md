## Introduction
Almost every, and maybe every firmware needs versioning. This is especially useful when the customer reports bugs and you need to understand what version is being used.

In this guide, I will try to show how you can assign a version to the firmware without having to worry too much about it

## Simple option

Of course, if you don't need to change the firmware version relatively often and you don't need additional information such as the commit hash or a test mark or something else, then you can simply use defines

```c
#define FW_VERSION_MAJOR        1
#define FW_VERSION_MINOR        0
#define FW_VERSION_PATCH        1
```

or

```C
#define FW_VERSION              "1.0.1"
```

However, it is often necessary to change the firmware version even before the release, and changing it manually can be stressful, for example, I always forget to do it. In such cases, it is useful to automate the firmware installation process, as well as add some additional information to facilitate the identification of the actual functionality of the firmware, as well as debugging. For automation, you will need ``git`` and its ``git tags``. Also, as a firmware version structure, we will use the following option:

``v[MAJOR].[MINOR].[PATCH]-[SHORT COMMIT HASH][DURTY BUILD]``


## Makefiles

For example, if your project is built using ``makefiles`` directly, the firmware definition in your makefile might look like this:

```makefile
VERSION_BUILD_TAG := $(subst v,,$(shell git describe --tags))
VERSION_HASH_SHORT := $(shell git rev-parse --short HEAD)

VERSION_MAJOR := $(word 1,$(subst ., ,$(VERSION_BUILD_TAG)))
VERSION_MINOR := $(word 2,$(subst ., ,$(VERSION_BUILD_TAG)))
VERSION_PATCH := $(word 3,$(subst ., ,$(VERSION_BUILD_TAG)))

UNCOMIMMITTED_CHANGES := $(shell git diff --name-only HEAD | wc -l)

ifeq ($(UNCOMIMMITTED_CHANGES), 0)
    DIRTY_BUILD_INDEX := ""
else
    DIRTY_BUILD_INDEX := "+"
endif

# Passing firmware version to compilation flags
CFLAGS += -DFW_VERSION_MAJOR=\"${VERSION_MAJOR}\" \
		  -DFW_VERSION_MINOR=\"${VERSION_MINOR}\" \
		  -DFW_VERSION_PATCH=\"${VERSION_PATCH}\" \
		  -DFW_VERSION_HASH=\"${VERSION_HASH_SHORT}\" \
		  -DFW_VERSION_DIRTY_INDEX=\"${DIRTY_BUILD_INDEX}\"
```

## CMake

But if the project is based on ``CMake``, then the firmware definition may look like this:

```cmake
# Getting the build tag of the firmware version without the 'v' prefix
execute_process(
    COMMAND git describe --tags
    OUTPUT_STRIP_TRAILING_WHITESPACE
    OUTPUT_VARIABLE VERSION_BUILD_TAG
)
string(REPLACE "v" "" VERSION_BUILD_TAG "${VERSION_BUILD_TAG}")

# Getting a short hash of the firmware version
execute_process(
    COMMAND git rev-parse --short HEAD
    OUTPUT_STRIP_TRAILING_WHITESPACE
    OUTPUT_VARIABLE VERSION_HASH_SHORT
)

# Extracting MAJOR, MINOR and PATCH versions from a build tag
string(REGEX MATCH "^[0-9]+" VERSION_MAJOR "${VERSION_BUILD_TAG}")

string(REGEX MATCH "[0-9]+\\.[0-9]+" TEMP_VERSION_MINOR "${VERSION_BUILD_TAG}")
string(REGEX MATCH "[0-9]+$" VERSION_MINOR "${TEMP_VERSION_MINOR}")

string(REGEX MATCH "[0-9]+\\.[0-9]+\\.[0-9]+" TEMP_VERSION_PATCH "${VERSION_BUILD_TAG}")
string(REGEX MATCH "[0-9]+$" VERSION_PATCH "${TEMP_VERSION_PATCH}")

# Check for unsaved changes
execute_process(
    COMMAND git diff --name-only HEAD
    OUTPUT_STRIP_TRAILING_WHITESPACE
    OUTPUT_VARIABLE UNCOMMITTED_CHANGES
)

# Counting the number of changes
string(LENGTH "${UNCOMMITTED_CHANGES}" UNCOMMITTED_CHANGES_COUNT)

# Setting DIRTY_BUILD_INDEX depending on the presence of unsaved changes
if(UNCOMMITTED_CHANGES_COUNT EQUAL 0)
    set(DIRTY_BUILD_INDEX "")
else()
    set(DIRTY_BUILD_INDEX "+")
endif()

# Output the firmware version for verification
message(STATUS "Firmware Version: ${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}-${VERSION_HASH_SHORT}${DIRTY_BUILD_INDEX}")

# Passing firmware version to compilation flags
add_compile_definitions(
    -FW_VERSION_MAJOR=\"${VERSION_MAJOR}\"
    -FW_VERSION_MINOR=\"${VERSION_MINOR}\"
    -FW_VERSION_PATCH=\"${VERSION_PATCH}\"
    -FW_VERSION_HASH=\"${VERSION_HASH_SHORT}\"
    -FW_VERSION_DIRTY_INDEX=\"${DIRTY_BUILD_INDEX}\"
)
```

## STM32CubeIDE or Eclipse based IDE

Of course, there are quite a lot of projects tied to a specific IDE and there is no possibility to flexibly configure makefiles. But even in such cases, you can automate the assignment of the firmware version. Since I mainly use STM32, therefore we will consider the option for ``STM32CubeIDE``, however, this option is also suitable for other IDEs.

Despite the fact that ``STM32CubeIDE`` has the ability to install ``Build Variables``, unfortunately they cannot be changed dynamically (or I didn't figure it out). Therefore, you can create a separate .h file that will be connected in the project itself.

First, create a ``bash`` file (and this is true for Windows systems as well), for example ``version.sh`` and save it in the ``Tools/`` directory, of course, you can save it wherever you like

```bash
#!/bin/bash

# Capture a tag and remove the v character
VERSION_BUILD_TAG=$(git describe --tags)
VERSION_BUILD_TAG=${VERSION_BUILD_TAG#v}

# Capture short hash
VERSION_HASH_SHORT=$(git rev-parse --short HEAD)

# Divide the tag into its components
VERSION_MAJOR=$(echo $VERSION_BUILD_TAG | cut -d'.' -f1)
VERSION_MINOR=$(echo $VERSION_BUILD_TAG | cut -d'.' -f2)
VERSION_PATCH=$(echo $VERSION_BUILD_TAG | cut -d'.' -f3)

# Check for uncommitted changes
UNCOMMITTED_CHANGES=$(git diff --name-only HEAD | wc -l)

if [ "$UNCOMMITTED_CHANGES" -eq "0" ]; then
    DIRTY_BUILD_INDEX=""
else
    DIRTY_BUILD_INDEX="+"
fi

# Prepare version.h file
FW_VERSION_FULL="v${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_PATCH}-${VERSION_HASH_SHORT}${DIRTY_BUILD_INDEX}"

# Export CubeIDE environment variable 
# (see Properties->C/C++ Build->Environment)
PWD=$(pwd)

cat << EOF > "$PWD/Core/Inc/version.h"
#ifndef __version_h
#define __version_h

#define FW_VERSION_FULL        "$FW_VERSION_FULL"
#define FW_VERSION_MAJOR       $VERSION_MAJOR
#define FW_VERSION_MINOR       $VERSION_MINOR
#define FW_VERSION_PATCH       $VERSION_PATCH

#endif /* __version_h */
EOF

```

And we include this file as ``Pre-build command``

<p align = "center">
	<img src="https://github.com/UladShumeika/How-to-determine-the-firmware-version/blob/main/setting_version.png" alt="Device operation example">
</p>


And of course, we connect this file to our project.

```C
#include "version.h"
```

## Сonclusion

This approach to assigning a firmware version is just one of many possibilities. There are certainly other methods, and you can adapt and expand on these techniques to fit different versioning strategies. I hope this information proves useful to you as you explore and implement the best practices for your projects.
