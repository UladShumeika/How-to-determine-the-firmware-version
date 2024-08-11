## Introduction
In the world of embedded software development, firmware versioning plays a crucial role. Even if the product rarely changes in its working state, during design and debugging, the software version can change rapidly. I don't know about you, but I often forget to update the version manually. Therefore, it's time to automate this process a bit.
In this guide, I describe the details I like to store in each build, how I capture these details, and how I make them available to the firmware.

There are various versioning options, depending on the project and the preferences of individual engineers. I would like to focus on the following format:
v[MAJOR].[MINOR].[PATCH]-[SHORT COMMIT HASH][DURTY BUILD].

Конечно же если вам не нужно относительно часто менять версию прошивки и вам не нужна дополнительная информация такая как HASH коммита, то можно использовать просто defines

```c
#define FW_VERSION_MAJOR        1
#define FW_VERSION_MINOR        0
#define FWVERSION_PATCH         1
```

или

```C
#define FW_VERSION              "1.0.1"
```

Однако довольно часто бывает необходимо менять версию прошивки еще даже до релиза, то изменять ее вручную бывает напряжно, например я всегда забываю это делать. В таких случаях полезно автоматизировать процесс установки прошивки, а также добавить немного дополнительной информации для облегчения индетификации фактической функциональности прошивки, а также отладки. Для автоматизации над понадобиться ``git`` и его ``git tags``. Также в качестве структуры версии прошивки будем использовать следующий вариант:

``v[MAJOR].[MINOR].[PATCH]-[SHORT COMMIT HASH][DURTY BUILD]``


## Makefiles

Например, если ваш проект построен на прямом использовании ``makefiles``, определение прошивки может выглядеть следующий образом:

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

## Cmake

А вот если проект, основан на ``Cmake``, то определение прошивки уже может выглядить вот так:

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

Конечно же есть достаточно большое количество проектов завязанных на конкретной IDE, так как я в основном использую STM32, поэтому рассмотрим вариант для ``STM32CubeIDE``, однако данный вариант подходит и для других IDE.

Несмотря на то, что в ``STM32CubeIDE`` есть возможность установить ``Build Variables``, к сожалению они не могут изменяться динамически (ну или я не разобрался). Поэтому можно создать отдельный .h файл который будет подключаться уже в самом проекте.

Сперва создаем ``bash`` файл (причем это верно и для Windows системы), например ``version.sh`` и сохраним в каталоге ``Tools/``, конечно же вы пожете сохранить его туда куда вам удобно

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

И подключаем данный файл как ``Pre-build command``

<p align = "center">
	<img src="https://github.com/UladShumeika/How-to-determine-the-firmware-version/blob/main/setting_version.png" alt="Device operation example">
</p>

![Alt text]("https://github.com/UladShumeika/How-to-determine-the-firmware-version/blob/main/setting_version.png")


 
