---
title: "From qmake to CMake: A Nontrivial Case Study"
subtitle: Cross-compiled and cross-platform build, deployment, and dependency management (using Conan)
layout: post
date: 2022-05-15
category: blog
author: amirmasoudabdol
headerImage: false
tags: CMake Qt6 QtDev C Cpp qmake R Rstats
description: A not so brief summary of the process of moving from qmake to CMake, and how to manage cross-compiled and cross-platform build, deployment, and depdendencies (using Conan)
image:
  path: /assets/posts/qmake-to-cmake-twitter-summary-card.png
  alt: Cross-compiled and cross-platform build, deployment, and dependency management (using Conan)
---

In this article, I will walk you through the process of rewriting JASP’s build system from scratch using CMake. It is a long journey of moving the entire build and deployment process from [qmake](https://doc.qt.io/qt-6/qmake-manual.html) to [CMake](https://cmake.org). In addition, I will discuss how we have used CMake to work with R framework, and how we have used Conan to manage our dependencies in order to achieve a robust cross-platform, multi-architecture build, and deployment. 

While this writing is very much focused on JASP project, and it might not directly apply to your project, I hope you can find bits and pieces of if helpful when you decide to go through the same process in your project. I should mention that I will not be covering every aspects of the software or the build process equally.

## About JASP, and Its Architecture

JASP is an open-source alternative to SPSS, SAS, and other lookalikes. It is built on top of the Qt Framework, and R. This means that the main body of the app is written in C++, i.e., GUI, Data Management, Engine. In contrast, all the statistical analyses are done in R, and the results are being communicated back to the GUI using JSON objects. JASP modules are specialized R packages consist of R, and some QML codes. The R part is responsible for calculation, and it is being executed in an R instance using the `JASPEngine`, and the QML part is being parsed by JASP to construct an ocean of check boxes[^1] and controls for the user to interact with! 

`JASPEngine` runs as a separate process alongside JASP, and uses `libR-Interface` ≅ `R` + `RInside` + `Rcpp` to run the R analyses and commands, and ultimately retrieve their results. The `libR-Interface` is a library developed by JASP Team. It encapsulates an interface to R, as well as some logic on how the two apps needs to communicate to each other.

> I am not intending to describe the entire architecture of JASP. My goal is to provide a high level overview of its architecture and components in order for you to be able to observe its complexity and hopefully find similar situations or solutions to/for your project.

<picture>
  <source srcset="/assets/posts/qmake-to-cmake-jasp-arch-dark.png" media="(prefers-color-scheme: dark)">
  <img src="/assets/posts/qmake-to-cmake-jasp-arch-light.png"  width="100%"/>
</picture>

### R Framework, R Packages and JASP Modules

JASP uses R as its computational back-end. Every command or analysis, one way or another, is being passed to an instance of R (via `JASPEngine`), and after a successful computation, the results are being communicated back to JASP's GUI using a JSON object (via `JASPEngine`). Upon the retrieval of results, JASP processes the results and generates a HTML page, and finally it displays it inside a Chromium WebEngine, i.e., [QtWebEngine](https://doc.qt.io/qt-6/qtwebengine-overview.html). 

#### R Framework

On both Windows, and macOS, a copy of R Framework is being embedded into JASP's binary so that JASP can communicate with R without needing to rely on system's R, or asking the user to install an instance of R. On Linux, JASP uses Flatpak to manage a local copy of R.

- **On Windows**, the entire R internal is being shipped. The `libR-Interface.dll` links itself against three main libraries (`R.dll`, `RInside.dll` and `Rcpp.dll`) to be able to talk to R, and performs its calculation. One complication that raises here is the fact that `libR-Interface.dll` needs to be linked against R libraries, and that means it needs to be compiled with GCC, or MinGW.[^2] At the same time, Qt 6 does not like to be compiled using GCC; therefore, we ought to deal with two compilers. Later on, I will explain this problem and discuss how I used CMake toolchains to overcome this issue.
- **On macOS**, the `R.framework` is being embedded into the Mac App Bundle, and `libR-Interface.dylib` dynamically links against `R.dylib` (a.k.a `R.framework`), `RInside.dylib` and `Rcpp.dylib`.[^3]

In both systems, I use CMake to glue everything together, more on this later.

> {% octicon info height:24 class:"color-fg-accent"%} I will be using "R Framework" to refer to the `R/` folder on Windows, and `R.framework` on macOS.


#### R Packages, and JASP Modules

R packages that are necessary to the function of JASP needs to be installed inside the R Framework and shipped within the final binary.[^4] Some of the R packages like [RInside](http://dirk.eddelbuettel.com/code/rinside.html) and [Rcpp](https://www.rcpp.org) are needed during the build, as they need to be linked against the `libR-Interface` and consequently `JASPEngine`; as a results they need to be treated differently. More on this later.

JASP modules are basically extended R packages with some QML files that are being used by JASP to construct the graphic user interface for each module, i.e., all those controls and checkboxes. In addition, they know how to prepare, and pass their results back to JASP. These modules need to be installed and shipped within the final binary, and they do not necessary need to be available during the build.

The process of installing both the R packages and JASP modules are being handled by CMake either during the configuration or at the build step, more on this later.

## Qt 5, qmake, and Dependencies

Prior to JASP 0.16.2, JASP was based on Qt 5, and it was using qmake as its build system generator. In order to manage the libraries, [a pool of pre-built libraries, headers, and frameworks](https://github.com/jasp-stats/jasp-required-files) for every platform and architecture was maintained by the developers, and in every build they were being manually linked to the final binary.

As we decided to upgrade to Qt 6 and CMake, I set to get rid of the manually maintained pool of libraries and headers. Fortunately, CMake is much better at dealing with third-party libraries than qmake. In addition, I have adapted [Conan](https://conan.io) as our package manager to be able to build the dependencies for every target individually.

## Qt6, CMake, and Conan

As of Qt 6, CMake is the [default build system](https://www.qt.io/blog/qt-and-cmake-the-past-the-present-and-the-future) generator of the Qt Framework. Qt team has started adapting the CMake in their internal processes, and projects and they are continuously improving their CMake toolsets in past years. As Qt 6 is becoming more mature, more and more [Qt-specific CMake commands](https://doc.qt.io/Qt-6/cmake-commands-qtcore.html) are being added, and the gap between qmake and CMake is getting smaller. 

> {% octicon alert height:24 class:"color-fg-attention"%} From here onward, I assume that you are familiar with CMake, Qt, and at least have read [this article](https://doc.qt.io/qt-6/cmake-get-started.html) and are familiar with basics of CMake. 

### CMake Setup

Now that we are somewhat familiar with the architecture of JASP, and its bits and pieces, let's start looking into how the project is setup, and what are the tasks that needs to be done.

```bash
jasp-desktop/
  ├── CMakeLists.txt
  ├── Common
  │    └── CMakeLists.txt
  ├── R-Interface
  │    └── CMakeLists.txt
  ├── Engine
  │    └── CMakeLists.txt
  ├── Desktop
  │    └── CMakeLists.txt
  ├── Modules/
  ├── Resources/
  ├── Tools
  │    └── CMake
  │        ├── R.cmake
  │        ├── Conan.cmake
  │        ├── Modules.cmake
  │        ├── Programs.cmake
  │        ├── Libraries.cmake
  │        ├── Dependencies.cmake
  │        └── ...
  ├── R (R/ or R.framework)
  └── conanfile.txt
```


As you can see, JASP consists of several sub-projects. In addition to the following four main sub-projects, we have the `Modules/` folder that contains the JASP Modules, and finally a generic `Resources/` folder.

- **Common**, leading to the generation of the `libCommon`
- **R-Interface**, which compiles into the `libR-Interface`. 
- **Engine**, which compiles into `JASPEngine` executable
- and **Desktop**, which is the main GUI of the app, and it mainly consists of Qt codes

Below you can see some of the main interactions and dependencies between all the entities of the project. I did not go into full length to describe whether the libraries are linked statically or dynamically; however, in general, 

- **on macOS** most libraries are linked statically except a few dependencies and Qt itself.
- **on Windows** most libraries are linked dynamically except `libCommon` and few other exceptions (if I recall correctly)

<picture>
  <source srcset="/assets/posts/qmake-to-cmake-jasp-dependencies-dark.png" media="(prefers-color-scheme: dark)">
  <img src="/assets/posts/qmake-to-cmake-jasp-dependencies-light.png"  width="100%"/>
</picture>
<figcaption class="caption text-small">JASP's Dependency Graph</figcaption>

### Dependencies 

Before we start plugging things together using CMake, we need to make sure that all our dependencies are ready and available. I categorize the dependencies into 3 main categories, 

- R Framework
- Dependencies that can be managed using Conan
- and dependencies that needs special care, e.g., [JAGS](https://mcmc-jags.sourceforge.io), [ReadStat](https://github.com/WizardMac/ReadStat)
  - These are dependencies that do not offer a versatile build script, or cannot be found in Conan's cellar

#### R Framework 

R Framework is a package consists of libraries, binaries, header files, help files, and anything else that needed for R to function. Its main library is `R.dll` (or `libR.dylib` on macOS) and its main executable is `R.exe` (or `bin/R` on macOS, see [here]({% post_url 2022-04-10-embedding-rframework-in-a-qt-mac-app-and-cross-compiling-for-two-architectures %}) for more on the anatomy of `R.framework`). As mentioned, `libR-Interface` and some other libraries in the JASP project will be linking to R Framework; therefore it needs to be available during the build process. 

In order to get the most recent version of the R Framework, and create a reproducible build, we use CMake to identify the architecture of the system, then download the R package, and place it inside the `build/`.

[`R.cmake`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Tools/CMake/R.cmake) is a CMake module written to manage this task. Based on the host/target system, it downloads the R package, unpack it, and place it in the `build/` folder. Additionally, if necessary, it downloads extra dependencies of R, e.g., Tk/Tcl, FORTRAN; and add them to the final package. Furthermore, it sets some of the R related parameters of the project.

##### On macOS

[On macOS](https://github.com/jasp-stats/jasp-desktop/blob/7c8c2a91fc914bc3a1c3b998611517d2fe0bdaf6/Tools/CMake/R.cmake#L98), a few key points needs to be dealt with:

- Setting path parameters 
- Downloading the right `R.framework` based on the host system.
- Downloading the corresponding FORTRAN compiler on the host system.
- Unpacking everything, and moving them into the `build/` folder.
- Patching the libraries, and resigning them using the Apple Developer Certificate
- Downloading and installation of `RInside` and `Rcpp` inside the `R.framework` for later usage, and linking. Read more [here]({% post_url 2022-04-10-embedding-rframework-in-a-qt-mac-app-and-cross-compiling-for-two-architectures %}).

Setting parameters:

```cmake
set(R_FRAMEWORK_PATH "${CMAKE_BINARY_DIR}/Frameworks")
set(R_HOME_PATH "${R_FRAMEWORK_PATH}/R.framework/Versions/${R_DIR_NAME}/Resources")
set(R_LIBRARY_PATH "${R_HOME_PATH}/library")
set(R_OPT_PATH "${R_HOME_PATH}/opt")
set(R_EXECUTABLE "${R_HOME_PATH}/R")
set(R_INCLUDE_PATH "${R_HOME_PATH}/include")
set(RCPP_PATH "${R_LIBRARY_PATH}/Rcpp")
set(RINSIDE_PATH "${R_LIBRARY_PATH}/RInside")
```

Declaring the `FetchContent` object for downloading the R Framework:

```cmake
fetchcontent_declare(
  r_pkg
  URL ${R_DOWNLOAD_URL}
  URL_HASH SHA1=${R_PACKAGE_HASH}
  DOWNLOAD_NO_EXTRACT ON
  DOWNLOAD_NAME ${R_PACKAGE_NAME})
```

Unpacking and moving the R.framework to the `build/` folder:

```cmake
execute_process(
  WORKING_DIRECTORY ${r_pkg_SOURCE_DIR}
  COMMAND tar -xf tcltk.pkg/Payload -C ${r_pkg_r_home}/)

execute_process(
  WORKING_DIRECTORY ${r_pkg_SOURCE_DIR}
  COMMAND tar -xf texinfo.pkg/Payload -C ${r_pkg_r_home}/)

make_directory(${CMAKE_BINARY_DIR}/Frameworks)
execute_process(
  WORKING_DIRECTORY ${r_pkg_SOURCE_DIR}
  COMMAND cp -Rpf R.framework ${CMAKE_BINARY_DIR}/Frameworks)
```

Installing the `RInside` and `Rcpp` inside the `R.framework`:

```cmake
file(
  WRITE ${MODULES_RENV_ROOT_PATH}/install-RInside.R
  "install.packages(c('RInside', 'Rcpp'), type='binary', repos='${R_REPOSITORY}', INSTALL_opts='--no-multiarch --no-docs --no-test-load')"
)

execute_process(
  ERROR_QUIET OUTPUT_QUIET
  WORKING_DIRECTORY ${R_HOME_PATH}
  COMMAND ${R_EXECUTABLE} --slave --no-restore --no-save
          --file=${MODULES_RENV_ROOT_PATH}/install-RInside.R)
```

##### On Windows

Similarly, [on Windows](https://github.com/jasp-stats/jasp-desktop/blob/7c8c2a91fc914bc3a1c3b998611517d2fe0bdaf6/Tools/CMake/R.cmake#L526), we need to download the R package, unpack it and place it inside the `build/` folder, and finally we need to install the `RInside` and `Rcpp` inside the `build/R` folder.

Downloading, unpacking and moving the R package to the `build/` folder:

```cmake
fetchcontent_declare(
  r_win_exe
  URL ${R_DOWNLOAD_URL}
  URL_HASH SHA1=${R_PACKAGE_HASH}
  DOWNLOAD_NO_EXTRACT ON
  DOWNLOAD_NAME ${R_PACKAGE_NAME})

execute_process(
  WORKING_DIRECTORY ${r_win_exe_SOURCE_DIR}
  COMMAND ${R_PACKAGE_NAME} /CURRENTUSER /verysilent /sp
          /DIR=${r_win_exe_BINARY_DIR}/R)

file(COPY ${r_win_exe_BINARY_DIR}/R DESTINATION ${CMAKE_BINARY_DIR})
```

Installing the `RInside` and `Rcpp` inside the `R/`:

```cmake
file(
  WRITE ${MODULES_RENV_ROOT_PATH}/install-RInside.R
  "install.packages(c('RInside', 'Rcpp'), type='binary', repos='${R_REPOSITORY}' ${USE_LOCAL_R_LIBS_PATH}, INSTALL_opts='--no-multiarch --no-docs --no-test-load')"
)

execute_process(
  ERROR_QUIET OUTPUT_QUIET
  WORKING_DIRECTORY ${R_BIN_PATH}
  COMMAND ${R_EXECUTABLE} --slave --no-restore --no-save
      --file=${MODULES_RENV_ROOT_PATH}/install-RInside.R)
```

> {% octicon alert height:24 class:"color-fg-attention"%} I am omitting a lot of details here; especially when it comes to how R packages installation works. As far as CMake concerns, you need to execute a process to call the `${R_EXECUTABLE}` and make sure that it can install your requested package.

##### On Linux

On Linux, since JASP uses Flatpak, we can rely on system's R; therefore we only need to located the R libraries, and set some paths for later usage, so that we can link everything and use the headers when needed.

#### Conan

Now that we have the R Framework ready to be used, and linked against. We need to deal with the rest of dependencies. Except a few libraries, all JASP dependencies can be found in ConanCenter, and therefore can simply be defined inside a [`conanfile.txt`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/conanfile.txt). 

```toml
[requires]
boost/1.78.0
libiconv/1.16
libarchive/3.5.2
zlib/1.2.11
zstd/1.5.2
jsoncpp/1.9.5
openssl/1.1.1m
bison/3.7.6
brotli/1.0.9

[generators]
cmake_paths
cmake_find_package

[options]
brotli:shared=True
```

While most of the time, running `conan install ..` inside the build directly is enough for setting up the libraries, we need to make sure that we can handle different architectures, especially on macOS where you can (need to) build both for `x86_64` and `arm64`. 

A savvy Conan user can often run the appropriate commands and set up the right environment, however, that's not always straightforward; therefore, I decided to delegate this task to a CMake module, [`Conan.cmake`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Tools/CMake/Conan.cmake). Here, based on the architecture, build type, and target properties', `Conan.cmake` decides what Conan command should be used:

```cmake
if(WIN32)

  message(STATUS "  ${CONAN_COMPILER_RUNTIME}")

  execute_process(
    COMMAND_ECHO STDOUT
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    COMMAND
      conan install ${CONAN_FILE_PATH} -s build_type=${CMAKE_BUILD_TYPE} -s
      compiler.runtime=${CONAN_COMPILER_RUNTIME} --build=missing)

elseif(APPLE)

  if(CROSS_COMPILING)

    execute_process(
      COMMAND_ECHO STDOUT
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
      COMMAND
        conan install ${CONAN_FILE_PATH} -s build_type=${CMAKE_BUILD_TYPE} -s
        os.version=${CMAKE_OSX_DEPLOYMENT_TARGET} -s os.sdk=macosx -s
        arch=${CONAN_ARCH} -s arch_build=${CONAN_ARCH} --build=missing)

  else()

    execute_process(
      COMMAND_ECHO STDOUT
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
      COMMAND
        conan install ${CONAN_FILE_PATH} -s build_type=${CMAKE_BUILD_TYPE} -s
        os.version=${CMAKE_OSX_DEPLOYMENT_TARGET} -s os.sdk=macosx
        --build=missing)

  endif()

endif()
```

Integrating Conan is often as simple as this. If you make sure that you are selecting the right libraries, and setting the right flags for your `conan install ..`, you can simply use `find_package` command to link your Conan libraries to your project. 

Notice that I have used two generators in the Conan file, `cmake_paths` and `cmake_find_package`. This is because some of those libraries are not properly set up to work with CMake projects, and therefore I need to use Conan environment variables to tap into their header or libraries folders. In addition, as we see later on during the Windows build, I needed a more fine-grained control over which libraries to select, so this was necessary.

#### Misc./Problematic Libraries

One of the problematic libraries that we had to deal with was ReadStat. ReadStat neither has support for CMake, nor (a proper) PkgConfig setup. This makes it challenging to integrate into a project, especially because we had to make sure that `libreadstat` is build correctly for our selected host and target. Therefore, we had to dive deeper into its native build system, and make sure that appropriate parameters are set. 

As you can see below, based on the system and CMake configurations, we instruct the `autoconf` to configure the ReadStat's Makefile accordingly. [`Dependencies.cmake`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Tools/CMake/Dependencies.cmake):

```cmake
set(READSTAT_CFLAGS
    "-g -O2 -arch ${CMAKE_OSX_ARCHITECTURES} -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET}"
)
set(READSTAT_EXTRA_FLAGS_1 "--with-sysroot=${CMAKE_OSX_SYSROOT}")
set(READSTAT_EXTRA_FLAGS_2 "--target=${CONFIGURE_HOST_FLAG}")
set(READSTAT_CXXFLAGS "${READSTAT_CFLAGS}")

add_custom_command(
  WORKING_DIRECTORY ${readstat_SOURCE_DIR}
  OUTPUT ${readstat_BINARY_DIR}/include/readstat.h
         ${readstat_BINARY_DIR}/lib/libreadstat.a
  COMMAND
    export CFLAGS=${READSTAT_CFLAGS} && export CXXFLAGS=${READSTAT_CXXFLAGS}
    && ./configure --enable-static --prefix=${readstat_BINARY_DIR}
    ${Iconv_FLAGS_FOR_READSTAT} ${READSTAT_EXTRA_FLAGS_1}
    ${READSTAT_EXTRA_FLAGS_2}
  COMMAND ${MAKE}
  COMMAND ${MAKE} install
  COMMENT "----- Preparing 'readstat'")

add_custom_target(readstat
                  DEPENDS ${readstat_BINARY_DIR}/include/readstat.h)

set(LIBREADSTAT_INCLUDE_DIRS ${readstat_BINARY_DIR}/include)
set(LIBREADSTAT_LIBRARY_DIRS ${readstat_BINARY_DIR}/lib)
set(LIBREADSTAT_LIBRARIES ${LIBREADSTAT_LIBRARY_DIRS}/libreadstat.a)
```

Similar situation raised when we started dealing with JAGS. With JAGS, on macOS, we had to provide an appropriate FORTRAN compiler as well, and make sure that the FORTRAN compiler as well is aware of the environment variables. We went to this length to make sure that our build is consistent and we can cross-compile our software for `x86_64` and `arm64`. [`Modules.cmake`](https://github.com/jasp-stats/jasp-desktop/blob/7c8c2a91fc914bc3a1c3b998611517d2fe0bdaf6/Tools/CMake/Modules.cmake#L457):

```cmake
fetchcontent_declare(
  jags
  URL "https://sourceforge.net/projects/mcmc-jags/files/JAGS/4.x/Source/JAGS-4.3.0.tar.gz"
  URL_HASH
    SHA256=8ac5dd57982bfd7d5f0ee384499d62f3e0bb35b5f1660feb368545f1186371fc
)

message(CHECK_START "Downloading 'jags'")

fetchcontent_makeavailable(jags)

if(jags_POPULATED)

  message(CHECK_PASS "successful.")

  set(JAGS_F77_FLAG "F77=${FORTRAN_EXECUTABLE}")
  set(JAGS_CFLAGS
      "-g -O2 -arch ${CMAKE_OSX_ARCHITECTURES} -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET}"
  )
  set(JAGS_EXTRA_FLAGS_1 "--with-sysroot=${CMAKE_OSX_SYSROOT}")
  set(JAGS_EXTRA_FLAGS_2 "--target=${CONFIGURE_HOST_FLAG}")
  set(JAGS_CXXFLAGS "${JAGS_CFLAGS}")

  add_custom_command(
    JOB_POOL sequential
    WORKING_DIRECTORY ${jags_SOURCE_DIR}
    OUTPUT ${jags_VERSION_H_PATH}
    COMMAND
      export CFLAGS=${READSTAT_CFLAGS} && export
      CXXFLAGS=${READSTAT_CXXFLAGS} && ${JAGS_F77_FLAG} ./configure
      --disable-dependency-tracking --prefix=${jags_HOME}
      ${JAGS_EXTRA_FLAGS_1} ${JAGS_EXTRA_FLAGS_2}
    COMMAND ${MAKE}
    COMMAND ${MAKE} install
    COMMAND
      ${CMAKE_COMMAND} -D
      NAME_TOOL_PREFIX_PATCHER=${PROJECT_SOURCE_DIR}/Tools/macOS/install_name_prefix_tool.sh
      -D PATH=${jags_HOME} -D R_HOME_PATH=${R_HOME_PATH} -D
      R_DIR_NAME=${R_DIR_NAME} -D
      SIGNING_IDENTITY=${APPLE_CODESIGN_IDENTITY} -D
      SIGNING=${IS_SIGNING} -D
      CODESIGN_TIMESTAMP_FLAG=${CODESIGN_TIMESTAMP_FLAG} -P
      ${PROJECT_SOURCE_DIR}/Tools/CMake/Patch.cmake
    COMMENT "----- Preparing 'jags'")
```

> {% octicon zap height:24 class:"color-fg-done"%} R is very particular about the FORTRAN compiler, and it uses `gfortran 8` on `x86_64` architecture, and `gfortran 12` on `arm64` architecture. To overcome this complication, `R.cmake` downloads the right FORTRAN, places it inside the `R.framework`, and makes sure that R toolchain can find and use it. On Windows, things are simpler as Rtools42 bundles everything R needs into a MSYS environment, and we can just use that instead. Also, the ARM Windows is not yet a thing!

### Managing the CMake Project

Now that we have all our dependencies ready, let's talk CMake project management. As shown below, we start by building each library separately using its own dedicated `CMakeLists.txt` configuration file.

```bash
jasp-desktop/
  ├── CMakeLists.txt
  ├── Common
  │    └── CMakeLists.txt
  ├── R-Interface
  │    └── CMakeLists.txt
  ├── Engine
  │    └── CMakeLists.txt
  ├── Desktop
  │    └── CMakeLists.txt
  ├── Modules/
  ├── Resources/
  ├── Tools
  │    └── CMake
  │        ├── R.cmake
  │        ├── Conan.cmake
  │        ├── Modules.cmake
  │        ├── Programs.cmake
  │        ├── Libraries.cmake
  │        ├── Dependencies.cmake
  │        └── ...
  ├── R (R/ or R.framework)
  └── conanfile.txt
```

The top level [`CMakeLists.txt`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/CMakeLists.txt) file starts with the project definition, and system variable detection routines, e.g., arch, build type. Besides that it mainly consists of several [`include`](https://cmake.org/cmake/help/latest/command/include.html) and [`add_subdirectory`](https://cmake.org/cmake/help/latest/command/add_subdirectory.html) commands loading various aspects of the project as you can see in the simplified version of the file below.

```cmake
# CMake Modules
###############

include(Config)
include(Conan)
include(Programs)
include(Libraries)
include(Dependencies)
include(JASP)
include(R)

# Sub Directories / Projects
############################

add_subdirectory(Common)

if(NOT WIN32)
  add_subdirectory(R-Interface)
else()
  add_custom_target(
    R-Interface
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/R-Interface
    BYPRODUCTS ${CMAKE_BINARY_DIR}/R-Interface/libR-Interface.dll
               ${CMAKE_BINARY_DIR}/R-Interface/libR-Interface.dll.a
    COMMAND
      ${CMAKE_COMMAND} -G "MinGW Makefiles" -S . -B
      ${CMAKE_BINARY_DIR}/R-Interface "-DMINGW_PATH:PATH=${MINGW_PATH}"
      "-DCMAKE_C_COMPILER:PATH=${MINGW_C_COMPILER}"
      "-DCMAKE_CXX_COMPILER:PATH=${MINGW_CXX_COMPILER}"
      "-DCMAKE_MAKE_PROGRAM:PATH=${MINGW_MAKE_PROGRAM}"
      "-DJASP_BINARY_DIR:PATH=${CMAKE_BINARY_DIR}"
      "-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}"
    COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}/R-Interface
    COMMENT "------ Configuring and Building the libR-Interface")
endif()

add_subdirectory(Engine)
add_subdirectory(Desktop)

include(Modules)

# Installation and Packing
##########################
 
include(Install)

if(NOT LINUX)
  include(Pack)
endif()
```

- The first part, **CMake Modules**, makes sure that dependencies are downloaded, prepared and if needed built on the *host system for the target system*. 
- The second part, **Sub Directories / Projects**, loads every sub project one by one and run their dedicated `CMakeLists.txt` and upon successful execution prepares the corresponding library or executable for the next subproject, e.g., `libCommon` is needed for successful creation of `libR-Interface`. In addition, the [`Modules`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Tools/CMake/Modules.cmake) CMake module installs all the JASP Modules.
  - [`Common/CMakeLists.txt`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Common/CMakeLists.txt)
  - [`R-Interface/CMakeLists.txt`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/R-Interface/CMakeLists.txt)
  - [`Engine/CMakeLists.txt`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Engine/CMakeLists.txt)
  - [`Desktop/CMakeLists.txt`](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Desktop/CMakeLists.txt)
- The third part, **Installation and Packing**, mocks the install process, deploys the project and prepares the it for the creation of the [WIX Installer](https://wixtoolset.org) or an [App Bundle](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/BundleTypes/BundleTypes.html) on macOS.

If you know the basic of CMake, you can see that nothing mysterious is happening in those `CMakeLists.txt` files; except maybe how some of those dependencies have been handled. There, I have utilized the [`cmake-generator-expressions`](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html) to link the appropriate libraries based on what is available where on different systems and/or different configurations. E.g., you can see below that `$<$<BOOL:${USE_CONAN}>:jsoncpp::jsoncpp>` instructs CMake to use `jsoncpp::jsoncpp` target if Conan has been used, i.e., on Windows and macOS; however, on Linux where we rely on system libraries, we use PkgConfig's variables to access the header files, and link to libraries, `$<$<PLATFORM_ID:Linux>:${_PKGCONFIG_LIB_JSONCPP_INCLUDEDIR}>`. 

```cmake
target_include_directories(
  Common
  PUBLIC # JASP
         jaspColumnEncoder
         # R
         ${R_INCLUDE_PATH}
         ${R_HOME_PATH}/include
         ${RCPP_PATH}/include
         #
         $<$<PLATFORM_ID:Linux>:${_PKGCONFIG_LIB_JSONCPP_INCLUDEDIR}>)

target_link_libraries(
  Common
  PUBLIC # jsoncpp
         $<$<BOOL:${USE_CONAN}>:jsoncpp::jsoncpp>
         $<$<PLATFORM_ID:Linux>:${_PKGCONFIG_LIB_JSONCPP_LIBRARIES}>
         $<$<PLATFORM_ID:Linux>:${_PKGCONFIG_LIB_JSONCPP_LINK_LIBRARIES}>
         #
         LibArchive::LibArchive
         # Boost
         Boost::filesystem
         Boost::system
         Boost::date_time
         Boost::timer
         Boost::chrono
         #
         $<$<BOOL:${JASP_USES_QT_HERE}>:Qt::Core>)
```

#### Building R-Interface on Windows, Rtools42, and Other Considerations for Windows

As briefly discussed, building `libR-Interface` on Windows is not a trivial task. This is due to the fact that the `libR-Interface` needs to be linked against `R.dll`, `RInside.dll` and `Rcpp.dll`. In addition, by adding their headers to the project, we trigger their preprocessors which are going to stop us from compiling if we are not compiling against a GCC-compatible compiler. 

As a result, we need to build the `libR-Interface` differently (using GCC) and link it dynamically against our other libraries and executable. Fortunately, or unfortunately, R team maintains their own MSYS system. This allows us to use GCC friendly compilers on Windows and build the `R-Interface.dll` there and finally links it against the JASP executable (+ Qt6) which is going to be built using MSVC.

In order to do this dance, we use the following code in the top-level `CMakeLists.txt`:

```cmake
if(NOT WIN32)
  add_subdirectory(R-Interface)
else()
  add_custom_target(
    R-Interface
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/R-Interface
    BYPRODUCTS ${CMAKE_BINARY_DIR}/R-Interface/libR-Interface.dll
               ${CMAKE_BINARY_DIR}/R-Interface/libR-Interface.dll.a
    COMMAND
      ${CMAKE_COMMAND} -G "MinGW Makefiles" -S . -B
      ${CMAKE_BINARY_DIR}/R-Interface "-DMINGW_PATH:PATH=${MINGW_PATH}"
      "-DCMAKE_C_COMPILER:PATH=${MINGW_C_COMPILER}"
      "-DCMAKE_CXX_COMPILER:PATH=${MINGW_CXX_COMPILER}"
      "-DCMAKE_MAKE_PROGRAM:PATH=${MINGW_MAKE_PROGRAM}"
      "-DJASP_BINARY_DIR:PATH=${CMAKE_BINARY_DIR}"
      "-DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}"
    COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR}/R-Interface
    COMMENT "------ Configuring and Building the libR-Interface")
endif()
```

Here, we tap out of our main CMake build process, and start a new process based on the `R-Interface/CMakeLists.txt` which itself is constructed such that it behaves as a standalone CMake projects on Windows, see below. By asking the CMake to construct a "MinGW Makefiles", and setting all the necessary paths to MINGW's C and C++ compiler, we initiate the configuration and build process of the `R-Interface` project from our main CMake project. If the build is successful, the `build/R-Interface` will contains the `libR-Interface.dll` and it is ready to be used by the rest of the project. 

Note that in the snippet above we are passing several of our local variables to the `${CMAKE_COMMAND}` when we are initiating the config/build of the `libR-Interface`. This is because when you are taping out of the main CMake process, the new process does not have access to your local variable anymore and you need to provide them to the other project manually, or using a template file.

```cmake
if(WIN32)

  set(QT_CREATOR_SKIP_CONAN_SETUP ON)

  cmake_minimum_required(VERSION 3.21)

  project(
    R-Interface
    VERSION 11.5.0.0
    LANGUAGES C CXX)

  list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/../Tools/CMake")

  # More CMake commands for building the R-Interface

else()

  # Building for macOS, and Linux without defining a new projects

endif()
```

It is worth noting that there is a lot of detailed involved in here, and if you are interested you should study the `R-Interface/CMakeLists.txt`, `Programs.Cmake` and the main `CMakeLists.txt` file. One of the complication that we faced was the fact that `libR-Interface.dll` needs to be linked against Boost, and it could not be linked against the Boost that was already prepared by Conan for the main project using MSVC. Instead, it needed to be linked against a Boost that is build inside the MSYS (i.e., Rtools42) environment. Therefore, we had to make sure that Boost (and jsoncpp) is available inside the MSYS system. Since Rtools42 is a full-featured MSYS environment, we can achieve this by installing those packages using `pacman`. After that, we only had to make sure that a copy of all those libraries sit next to the `JASP` executable because Windows. So, some manual (via CMake's `file` command) copy/paste of libraries were needed.

> {% octicon alert height:24 class:"color-fg-attention"%} I have mentioned Rtools42, but at the time that we implemented this method, R 4.0 was not released yet. Therefore, we had to use the combination of Rtool35 and MSYS to achieve a complete build. Most links to CMake files in this article are pointing to [`tags/0.16.2`](https://github.com/jasp-stats/jasp-desktop/tree/v0.16.2). If you are interested on how Rtools42 has simplified this process, you can study this pull request, [Support for R-4.2.0](https://github.com/jasp-stats/jasp-desktop/pull/4763). In this RP, I have replaced the MSYS environment with Rtools42. (The PR is closed, however, it is content is merged into this [pull request](https://github.com/jasp-stats/jasp-desktop/pull/4801).)

### Build

Now that all our dependencies, libraries, and packages ready, we should be able to simply run the CMake command, and compile everything. I am not going to go into details on how to initiate the build, and I assume that you are familiar with the following sequence, or, you are going to open the project in Qt Creator, and rely on its integration! 🤞🏼

```
mkdir build && cd build
cmake .. -GNinja -DCMAKE_PREFIX_PATH=<path-to-qt>
ninja
```

### Installation and Deployment

The one last thing to do is to deploy the project. I will not go into details of the deployment, rather I mention a few common issues or obstechales that we had to deal with, and leave you with a link to [Install](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Tools/CMake/Install.cmake) and [Pack](https://github.com/jasp-stats/jasp-desktop/blob/v0.16.2/Tools/CMake/Pack.cmake) modules.

Deployment of Qt projects are done using the `macdeployqt` and `windeployqt`. Unfortunately, neither of these tools are robust enough to do everything and you will end up having to do several things manually. [CPack](https://cmake.org/cmake/help/latest/module/CPack.html) is designed to help you with the deployment of your project; however, to take advantages of the CPack, you must have a very standard project tree, libraries, and dependencies. In our case, the `R.framework` and exceptions around the `libR-Interface` did not allow us to use CPack successfully. 

- **On macOS**, CPack can construct a `.app` bundle form your project; however, you have a slim chance of getting this to work if you are using an external Framework, in our case `R.framework`. 
- **On Windows**, we have already had a WIX template and we still ran into problem due to the complication caused by `R/` and some of the modules. 

In both system, our approach was to stage the installation and fully deploy the project, and all its libraries inside the `Install/` folder, and pass everything to WIX on Windows, and on macOS put everything precisely where they need to be according to [Apple's Bundle Structure](https://developer.apple.com/library/archive/documentation/CoreFoundation/Conceptual/CFBundles/BundleTypes/BundleTypes.html#//apple_ref/doc/uid/10000123i-CH101-SW1).

The [`install`](https://cmake.org/cmake/help/latest/command/install.html) command is your friend. Set up your project structure, and handle as much as your installation logic using the `install` command, e.g., 

```cmake
set(MACOS_BUNDLE_NAME JASP)
set(JASP_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}/${MACOS_BUNDLE_NAME}.app")
set(JASP_INSTALL_BINDIR "${JASP_INSTALL_PREFIX}/Contents/MacOS")
set(JASP_INSTALL_RESOURCEDIR "${JASP_INSTALL_PREFIX}/Contents/Resources")
set(JASP_INSTALL_FRAMEWORKDIR "${JASP_INSTALL_PREFIX}/Contents/Frameworks")
set(JASP_INSTALL_MODULEDIR "${JASP_INSTALL_PREFIX}/Contents/Modules")
set(JASP_INSTALL_DOCDIR "${JASP_INSTALL_RESOURCEDIR}")

install(
  TARGETS JASP JASPEngine
  RUNTIME DESTINATION ${JASP_INSTALL_BINDIR}
  BUNDLE DESTINATION .)

install(
  DIRECTORY ${_R_Framework}
  DESTINATION ${JASP_INSTALL_FRAMEWORKDIR}
  REGEX ${FILES_EXCLUDE_PATTERN} EXCLUDE
  REGEX ${FOLDERS_EXCLUDE_PATTERN} EXCLUDE
  REGEX ".*/bin/R(Script)?$" EXCLUDE
  )
```

### {% octicon alert height:24 class:"color-fg-danger"%} Caveats 

As mentioned before, due to the requirements of JASP project, we had to deal with several type of dependencies. Some managed by Conan, others fetched using the `FetchContent`, and the rest were downloaded directly from their repository. I have to emphasize that this is not an ideal scenario, and you should avoid it if you can. You should try to manage all your dependencies with one dependency manager. In our case, we could have improved our setup by writing a Conan recipe for ReadStat, and let Conan handle its build as well. However, we could have not moved everything to Conan, e.g., R Framework and JAGS (due to its FORTRAN dependency). 

Additionally, I should reiterate the fact that, *ideally* you should not interact with Conan within your CMake files, and educate your team to use Conan command line instead. However, in our case, due to time pressure and because the team was not familiar with Conan, I decided to manage some of the Conan parameters with CMake. This is not ideal because you risk replying on deprecated Conan features' as Conan evolves.[^7] In our case, I have made a note/task of this fact and encouraged the team to deprecate the `Conan.cmake` module, study Conan, and setup their environment manually for each build.

## Summary

CMake offers a lot more freedom compared to qmake. I am aware that CMake is not everyone's cup of tea and many people do not like it. However, I feel if you have tried CMake before and had similar experience, you need to give it another try. A lot has changed from early days, and large and complicated projects like Qt or KDE heavily rely on it. In Modern CMake[^5], you have much more control over your targets, and libraries. CPack, although not perfect, can facilitate your deployment greatly if you have a relatively standard project. Finally, dependency managers like Conan are improving the CMake integration every day, and can generate CMake config files for packages that do not come with CMake support of the box.[^6] 

I should also reiterate that most of what I have discussed might not directly apply to your project; however, I hope this could come handy if you are trying to resolve some of your build system issues', or rewriting it from scratch because you want to move away from qmake to CMake. Or at very least, I hope it gives you an idea on how far you can get with CMake when it comes to a rather complicated and irregular project, which is about every C/C++ project out there! 😅

*P.S. I was expecting this article to become long, but not this long! Feel free to reach out on [Twitter](https://twitter.com/amirmasoudabdol), or [Cpplang Slack](cpplang.slack.com) where I occasionally spend some time, and try to learn from very knowledgeable members of `#conan`, `#cmake` and `#qt` channels!* 🙏🏼


[^1]: [If OpenSSL were a GUI](https://smallstep.com/blog/if-openssl-were-a-gui/)
[^2]: [R](https://www.r-project.org/about.html) is a [GNU Project](http://www.gnu.org) and therefore it can only be compiled using the [GNU GCC](https://gcc.gnu.org) compiler. This means, anything that needs to be linked directly to R libraries needs to use a GCC-compatible compiler.
[^3]: I have already covered the complicated and cumbersome process of successfully [embedding the `R.framework` into the app bundle]({% post_url 2022-04-10-embedding-rframework-in-a-qt-mac-app-and-cross-compiling-for-two-architectures %}), as well as the process of signing and notarizing the app bundle for distribution. 
[^4]: Most of the R packages are being installed via [Renv](https://rstudio.github.io/renv/articles/renv.html).
[^5]: I highly recommend [Professional CMake: A Practical Guide](https://crascit.com/professional-cmake/) book by Craig Scott. I personally have learned a lot from it. You can often find him answering some question on [CMake Discourse](https://discourse.cmake.org) as well.
[^6]: The quality of the config file you get out of Conan depends on the quality of the Conan script but in general, chances are high that you get a good, and working config file, so that you can simply use the target, e.g., `jsoncpp::jsoncpp`.
[^7]: [Conan 2.0](https://docs.conan.io/en/latest/conan_v2.html)