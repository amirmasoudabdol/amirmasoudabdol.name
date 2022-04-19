---
title: "Embedding R.framework in a Qt Mac App, and Cross Compiling for Two Architectures"
layout: post
date: 2022-04-18
category: blog
author: amirmasoudabdol
image: /assets/posts/embedding-r-framework-twitter-summary-large-image.png
projects: false
description: How to Bundle the R.framework with Your macOS App
hidden: false # don't count this post in blog pagination
---

<picture><source srcset="/assets/posts/embedding-r-framework-dark.png" media="(prefers-color-scheme: dark)"><img src="/assets/posts/embedding-r-framework-light.png" width="50%" align="right" float="right"/></picture>
Developing for Apple platform *can be* fun if your app lives solely inside Xcode, and you only utilize standard libraries, or don't interact with any alien libraries or frameworks. In that case, there is a high chance that you have never been worried about compatibility issues, linking problems, signing struggles and the notarization*!* 

However, if your software is built on top of a different GUI framework rather than UIKit, e.g., Qt Framework, and you need all sort of other libraries; then you are familiar with the pain of getting them to know each other, link together properly, and build into one coherent piece of software. Finally, you are going to prepare your App Bundle, sign it, and send it to Apple for notarization and start praying! Now, add the **`R.framework`** or any other non-standard framework to the mix, and you are very close to the definition of nightmare.[^1]

JASP is one of these software, where in addition to embedding the `R.framework`, we include series of R packages and Modules. Some can be downloaded as binaries from CRAN, some need compilation, and some needs third-party libraries, e.g. [JAGS](https://sourceforge.net/projects/mcmc-jags/). Now, add to all that the fact that we need to do all that for two architectures, i.e., `arm64` and `x86_64`, and three platforms.

In this article, I go through some of the challenges that we had to overcome to get all this to working in a stable manner on macOS. Most of the heavy-lifting of the build system is being taken care of by CMake which is not discussed here; instead, in this post I focus on preparing and embedding the `R.framework`.

## R.framework

`R.framework` is inherently not portable. This means the entire Framework is built with hardcoded paths. It expects to be installed in `/Library/Frameworks/R.framework`, and every other libraries and packages expect it to be there so that they can link to the R libraries and therefore function correctly. Placing the `R.framework` to any other location on your system is merely asking for trouble. One of the biggest consequences of installing `R.framework` in a non-standard place is that none of the CRAN binary packages can navigate their ways to R's home directory, consequently they cannot find the `libR.dylib` and therefore they won't work.

For us, this is a big issue as we need to embed the `R.framework` to our app such that it can sit and function inside the **JASP.app** and work in our user's system. In addition, we need to download several pre-build R packages from CRAN and inform them about the whereabouts of `libR.dylib`. Finally, we need to make sure that JASP modules can be built on our build machines and work inside our app bundle.

### Anatomy of R.framework

`R.framework`, i.e., R, is a complicated and busy framework. It is designed to comply with [Apple's Framework Bundles Structure](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFrameworks/Concepts/FrameworkAnatomy.html), but underneath its Apple's flavored anatomy, everything is organized according to a standard [GNU Install Structure](https://www.gnu.org/software/automake/manual/1.4/html_node/Install.html) where binaries lives in `bin/` and headers are in `include/`, etc. This means when you are inside the GNU environment things get Linux-y as much as they could be on a Darwin subsystem.

Apple's Framework Bundle is wrapper around a dynamic library, and all the miscellaneous files needed by the library. In the case of the `R.framework`, everything from include files, fonts, libraries and modules are included in the Framework. One crucial aspect of designing a Framework is to make sure that components of the framework can reliably find each other. In `R.framework`, this is insured by having everything route through R's home directory, defined by the `R_HOME` environment variable. If you install the `R.framework` in its default place, `R_HOME` is set to `/Library/Frameworks/R.framework/Resources` which is a symbolic link to `/Library/Frameworks/R.framework/Versions/A/Resources` where `A` is the version of the R that you are using, e.g., `4.1` or `4.1-arm64`. 

Here, we can see that everything is neatly placed in their respectable folder:

```
Frameworks/R.framework/Versions/4.1-arm64/Resources
├── bin
  ├── exec
    └── R
  └── R.sh
├── doc
├── etc
  ├── ldpaths
  └── Makeconf
├── fontconfig
├── include
├── lib
├── library
├── man1
├── modules
├── opt
├── share
└── tests
```

Inside the `bin/` folder, you find `exec/R` which is the actual R executable. However, this file is not simply being called by its own, instead, it is called from a bash script, `bin/R.sh`. This script does a lot of work, and one of them is to set the `R_HOME` variable and to inform the `bin/exec/R` about it. 

Another crucial step that `bin/R.sh` takes is to load the `etc/ldpaths`. This file then makes sure that `bin/exec/R` and all the other libraries needed during the execution of R can be found by the system during the loading and execution process. If you are familiar with the concept of library pathing, then `ldpaths` sets the `DYLD_FALLBACK_LIBRARY_PATH` so that when `bin/exec/R` requests a library, the system knows where to look for them. 

Another important file is placed in `etc/` folder, `etc/Makeconf`. This file sets several environment and R variables, from compiler flags, library paths, header paths, to path of the Fortran compiler. These are useful and necessary information for when you are requesting R to build a package from source. More on this later.

## Embedding R.framework

In order to make the framework portable, we need to make a few changes to the `bin/R.sh`. First, `R_HOME` has to correctly point to our desired path, second, we need to make sure that `bin/exec/R` and all the other libraries inside the `R.framework` can find each other during the loading process.

On the very top of the `bin/R.sh`, we need to set the `R_HOME_DIR` to *our* custom R's home directory path. Then, I set the following variables as follow

```bash
R_HOME="${R_HOME_DIR}"
R_SHARE_DIR=${R_HOME_DIR}/share
R_INCLUDE_DIR=${R_HOME_DIR}/include
R_DOC_DIR=${R_HOME_DIR}/doc 
```

If you are planning to simply run R from a different place in your system, and do not want to install any packages, you are done! However, if you are planning to install any other packages, and later place the `R.framework` inside an App Bundle, your journey is just starting*!* 

Your first step is to disable the default `LD_PATH` that is being set in `etc/ldpaths`. You can do this by commenting the following line so that `bin/R.sh` doesn't try to overwrite/set the library path for you. While this is not strictly needed, it gives you more granular controls over your library pathing and simplifies your debugging later on.

```bash
# . "${R_HOME}/etc${R_ARCH}/ldpaths"
```

After this, if you try to run the `bin/R.sh`, you notice that the executable expectedly won't work because system cannot find the libraries requested by the R executable. 

Now, in the rest of this post, we are going to take care that libraries inside the `R.framework` can find each other even though we have disabled the `ldpaths`; in addition, we are going to modify the CRAN binaries upon installation and replace their hard-coded paths. To be able to follow the upcoming section, you need to be familiar with the concept of `RPATH` and `LD_LIBRARY_PATH`.[^2]

### Patching Library Path and `install_name_tool`

At this point, we know that `bin/R.sh` doesn't know where `libR.dylib` and other libraries are. You can check the internal of a binary file using the `otool` command on macOS. Running `otool -L bin/exec/R` lists all the libraries that are linked to the executable:

```
❯ otool -L bin/exec/R
exec/R:
	/Library/Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libR.dylib (compatibility version 4.1.0, current version 4.1.3)
	/Library/Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libRblas.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1311.0.0)
```

We see that `bin/exec/R` is looking for `libR.dylib` in the default path `/Library/Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/` which is in fact `${R_HOME}/lib` but since we have changed it, it will not be able to find any of them, and will be terminated before it can get far into its execution process.[^3]

After reading the above footnote, and refreshing your `LD_LIBRARY_PATH` knowledge, you now know how everything works, and that we are going to use the `install_name_tool` to change the library path*!* 😬

Since we are going to embed the `R.framework` into an App Bundle, we are going to use the `@executable_path` variable as our anchor point. Our executable is located in `JASP.app/Contents/MacOS/JASP` and our embedded `R.framework` will be placed in `JASP.app/Contents/Frameworks/R.framework`, so, by changing all `/Library/Frameworks` prefix with `@executable_path/../Frameworks` we make sure that if `JASP` executable calls the R executable, libraries can be found in the relative path to the main binary, ie. `JASP`, therefore

```
❯ otool -L bin/exec/R
exec/R:
	@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libR.dylib (compatibility version 4.1.0, current version 4.1.3)
	@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libRblas.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1311.0.0)
```

However, this is not yet enough. If we are in `bin/exec/` and we try to run the already modified R executable, the `@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libR.dylib` does not lead to anything. To overcome this issue, we need to create a symlink to the Frameworks folder inside the `${R_HOME}/bin/` that points to `JASP.app/Contents/Frameworks`. By doing so, when R executable is being called, and system starts looking for `libR.dylib` in `@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libR.dylib` there will be a symlink that leads to the location of the `libR.dylib`. Basically, we are making sure that `Frameworks` folder is placed in the same relative location to R executable, and `JASP` executable, and `@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libR.dylib` is always well-defined regardless of which executable is being called.

```
├── bin
  ├── Frameworks -> ../../../../../../Frameworks
  ├── exec
    └── R
  ├── .
  ├── .
  ├── .
  └── R
├── lib
    ├── libR.dylib
    ├── libRblas.dylib
```

Now, you need to do the same things for all the libraries that are inside the `lib/` folder.

```
❯ otool -L libR.dylib
libR.dylib:
	@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libR.dylib (compatibility version 4.1.0, current version 4.1.3)
	@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libRblas.dylib (compatibility version 0.0.0, current version 0.0.0)
	@executable_path/../Frameworks/R.framework/Versions/4.1-arm64/Resources/lib/libgfortran.5.dylib (compatibility version 6.0.0, current version 6.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1311.0.0)
	/System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 1853.0.0)
	/usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
	/usr/lib/libbz2.1.0.dylib (compatibility version 1.0.0, current version 1.0.8)
	/usr/lib/libz.1.dylib (compatibility version 1.0.0, current version 1.2.11)
	/usr/lib/libicucore.A.dylib (compatibility version 1.0.0, current version 68.2.0)
	/usr/lib/libiconv.2.dylib (compatibility version 7.0.0, current version 7.0.0)
```

### Dealing with Other Libraries in `library/`, `modules/` and `opt/`

If you look around the `R.framework`, you will notice that there are a lot more libraries that all need to be patched with the same procedure. Most libraries are linked against `libR.dylib` and other libraries located in `R_HOME/lib`, so replacing the `/Library/Frameworks/` with `@executable_path/../Frameworks` will resolves a lot of issues. However, it does not cover all the edge cases. At JASP, we use a [CMake script](https://github.com/jasp-stats/jasp-desktop/blob/stable/Tools/CMake/PatchR.cmake) to make sure that all libraries are patched.

### CRAN Packages

When you are installing a CRAN binary package, you are requesting a pre-built version of the library. As discussed, this means, for instance, `MASS.so` will except to find the `libR.dylib` in `/Library/Frameworks/Resources/lib`. If you have the `R.framework` at its preferred place, every CRAN binary package will find the R library by following the `R_HOME`.

However, if `R.framework` is sitting in a non-default path, every CRAN binary package comes with a wrong library path; therefore, you have to patch every single of them manually as you did for all the first-party libraries and modules, as described before.

### macOS Security and Signing

Modern macOS is quite strict about the identity and origin of the binaries and libraries that are allowed to run in a system. Every Framework or App needs to be signed, verified and notarized by Apple and it’s then and only then that macOS will fully trust and run them without complaining.[^This was a big issue for us where at some point due to the complexity of `JASP.app`, we couldn't continue notarizing our app with Apple, so our users ended up getting scary warnings when they tried to run the app.]

`R.framework` comes pre-signed and notarized; however, any tempering with its guts will avoid its notarizations and signature. In fact, everything that we did so far falls into the definition of tempering. On a Mac with all default security options enabled, the modified `bin/exec/R` will not run, and macOS refuses to verify any of the libraries, e.g., `libR.dylib`, and therefore terminates the R executable at runtime. 

Fortunately, we can resign a binary/library, and get around this issue. This means that you need to remember to sign every single binary/library that you are modifying during this process. For us, at JASP, we need to sign every libraries on the fly as we build our software. This is because at the same time where we are building our Desktop app, we are also calling the R executable to install our Modules inside the `R.framework`. You can see part of this process [here](https://github.com/jasp-stats/jasp-desktop/blob/stable/Tools/CMake/Patch.cmake) where we are using a CMake module to automate this process during the configuration and build stages.

### CMake, Qt, R.framework, and Cross Compilation

<img src="/assets/posts/intel-and-apple-silicon.png" style="float:right; width: 50%;" />

Cross compilation is the process of compiling a program for different architectures and platforms. Most C/C++ compilers are capable of cross compiling, and most build system and build system generators, e.g., [automake](https://www.gnu.org/software/automake/manual/html_node/Cross_002dCompilation.html), [CMake](https://cmake.org), simplify the cross compilation by providing higher level flags, or variables. For instance, when using CMake on macOS, setting the `MACOSX_DEPLOYMENT_TARGET` variable ensures that the compiler builds your program for the specific architecture. 

JASP core is written in C++, and we are using Qt Framework and other C++ libraries; and on top of that, CMake is used to generate the build system command; therefore, we can target different architecture and version by configuring CMake correctly. However, `R.framework` as an external pre-built framework doesn't have any ideas about CMake, and it is only being linked to JASP App Bundle. The other problem is the Fortran compiler which is not fully stable on Apple Silicon; therefore its cross compilation is not to be trusted. 

In the rest of this post, I briefly walk through the steps necessary for treating the `R.framework` such that it can correctly cross compile R source packages.

### Cross Compilation of R Source Packages

With JASP 0.16.2, we rewrote our build system from scratch, see [here](https://github.com/jasp-stats/jasp-desktop/pull/4658). We committed to this effort to resolve some of the underlying issues in our backend. In addition, we wanted to have a native version of JASP for Apple Silicon, and that meant we had to deal with two different architectures, two different `R.frameworks` and a whole lot of other details. I am planning to write another post about this transition; here though, I would like to discuss what this cross compilation mean for `R.framework` and the process of embedding it to JASP.

Everything that I have mentioned above applies to both versions of `R.framework`, ie., `x86_64` and `arm64`. Libraries and binaries should be patched similarly, the resigning procedure is as important and it should be done on the fly; and `bin/R.sh` and `R_HOME` need attentions. `R.framework` in different architectures do a good job of downloading the right binary from CRAN; so, if you have already taken care of the library paths, you embed the right `R.framework` to the right `JASP.app`, either `x86_64` or `arm64`, and you can install R packages and you are guaranteed to get the right binary. 

However, the problem arises when you have to compile something! Since `R.framework` does not come with its own compiler toolchain, you need to make sure that during the cross compilation, R's internal are aware of your compiler flags.

#### Notes on `Makeconf` and `fortran`

As mentioned, CRAN is doing a great job of providing binary packages, and the only thing you need to do is to patch and resign your downloaded libraries, and make sure that R can find them. However, sometimes, you need to build a package, in our case our JASP Modules. This means that R needs to have access to compilers, and development toolchains. 

On macOS, you can get your C/C++ compiler by downloading and installing Xcode, and after that, R can use system variables and find the compiler with no problem. However, Xcode doesn't ship with `gfortran` and you often cannot use the Fortran that comes with Homebrew as you may ran into [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) incompatibilities, even if your build passes. In addition, `R.framework` is very particular about the `gfortran`. So, what you want to do is to download the right Fortran, install it somewhere, and make sure that R can find it. This is a problematic step even without making R portable, or cross-compilation attempt. Even worst, on Apple Silicon, the Fortran is not yet stable, so you need to go with [an experimental version](https://github.com/fxcoudert/gfortran-for-macOS).

Since we are cross compiling, we need to have a Fortran compiler that is capable of cross compilation; however, considering the way R uses the Fortran compiler, at this moment, I would not recommend using the Fortran compiler for cross compilation. Instead, we need to download two versions of Fortran, ie., `x86_64` and `arm64`, and link the right compiler to the right `R.framework`. 

Now, the problem is how to inform the `R.framework` about the location of Fortran.

##### Enters `etc/Makeconf`

As mentioned, most of the compiler configurations, including the host system and `R.framework`'s build flags are set in `etc/Makeconf`. In order to guide the `R.framework` to the right `gfortran`, or any other tools, we need to modify the `etc/Makeconf` file, and that's what we are doing at JASP. During the configuration step, we set the target architecture, ie., `x86_64` and `arm64`, download the right Fortran compiler, unpack and patch it; and finally, modify the `etc/Makeconf` such that the `FC` variable points to the right `gfortran` executable.

To be precise, we download the `gfortran` installer, unpack it inside the `R.framework`, e.g., `${R_HOME}/opt/local/gfortran` or `${R_HOME}/opt/R/arm64/gfortran` and make sure that any references to the Fortran compiler (in `etc/Makeconf`) points to the right path, e.g.,

```bash
sed -i.bak -e "s/\\/opt\\/R\\/arm64/$(R_HOME)\\/opt\\/R\\/arm64/g" Makeconf
```

This guarantees that any R packages that needs building is being build for the target architecture, even if it needs to compile Fortran code.

## Summary

In this post, I have tried to *briefly* summarize some of the most important steps needed to successfully patch and embed the `R.framework` into a macOS App Bundle. I mainly focused on the process of preparing the `R.framework`, making sure that it can work from a non-default location, patching its executable and libraries, and modifying the CRAN binary packages such that they can live inside the altered framework. In addition, I touched upon the topic of cross compilation, and what needs to be done when it comes to R source packages.

I hope that in the next post, I can dig deeper into the role of CMake, and explain how we utilize CMake tools and capabilities to glue different aspects of the build together. In the meantime, if you are struggling with `R.framework` please feel free to reach out on Twitter, or write to me. I’d be happy if I could be of any help*!*

[^1]: Embedding `R.framework` is such a cumbersome task that as far as I can tell, even the RStudio team gave up on it, and simply ask you to download the `R.framework` if you don't have it.
[^2]: You can refresh your knowledge by reading [Shared Libraries: Understanding Dynamic Loading](https://amir.rachum.com/blog/2016/09/17/shared-libraries/#debugging-cheat-sheet).
[^3]: If you are not familiar with these concepts, you first need to read and understand a few things about how libraries are linked together, and how they can find each other if they are not placed in the same directory, see [here](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/OverviewOfDynamicLibraries.html#//apple_ref/doc/uid/TP40001873-SW1) and [here](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/RunpathDependentLibraries.html#//apple_ref/doc/uid/TP40008306-SW1).