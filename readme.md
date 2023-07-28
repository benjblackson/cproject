
[![Build status](https://github.com/lmapii/pkt/workflows/ci/badge.svg)](https://github.com/lmapii/pkt/actions)

# Drafting a C development environment using CMake, Docker, Unity, and GitHub Actions. <!-- omit in toc -->

Sometimes, `C/C++` projects have a long development cycle. When working on such a project, it can easily happen that as developers we take the development environment for granted, and forget about the efforts invested in its bring-up: The build environment works like *magic*, the test framework is neatly integrated, and the CI/CD pipeline relieves us of tedious, repetitive tasks.

For me, all it took was a simple thought: To develop a `C` library, consisting of only a handful of files, but developed out of the context of my current, comfortable development environment. And there I was, back in the cold reality of `C/C++`, where you have the freedom - or obligation - to choose the entire development environment yourself.

Maybe this little writeup helps the next person that is feeling a little overwhelmed when trying to start a new `C/C++` project from scratch, with their hands frozen over the keyboard, thinking about all those things that need to be done before even starting to write a single line of code...

## Contents <!-- omit in toc -->

- [A "modern" development environment](#a-modern-development-environment)
- [Containers for development](#containers-for-development)
  - [Why Docker? Where's the catch?](#why-docker-wheres-the-catch)
  - [Let's do this!](#lets-do-this)
  - [A shortcut for verbose command line invocations](#a-shortcut-for-verbose-command-line-invocations)
- [Visual Studio Code Dev Containers](#visual-studio-code-dev-containers)
- [A skeleton subsystem](#a-skeleton-subsystem)
- [Building outside of `vscode` and mounting volumes](#building-outside-of-vscode-and-mounting-volumes)
  - [A note on Docker volumes](#a-note-on-docker-volumes)
  - [Running the development container manually](#running-the-development-container-manually)
- [Installing `clang` tools for formatting and static code analysis](#installing-clang-tools-for-formatting-and-static-code-analysis)
  - [Installing specific in the builder image](#installing-specific-in-the-builder-image)
  - [Using `clang-format` in the Development Container](#using-clang-format-in-the-development-container)
  - [Wrapping `clang-format` and `clang-tidy` calls](#wrapping-clang-format-and-clang-tidy-calls)
- [Adding unit tests](#adding-unit-tests)
  - [Installing `Unity` and `Ceedling` in the builder image](#installing-unity-and-ceedling-in-the-builder-image)
  - [Configuring `Unity` and running unit tests](#configuring-unity-and-running-unit-tests)
  - [Bonus: Coverage reports](#bonus-coverage-reports)
- [Getting started with GitHub workflows](#getting-started-with-github-workflows)
  - [Adding a GitHub action](#adding-a-github-action)
  - [Building an image in a GitHub action](#building-an-image-in-a-github-action)
  - [Using caches to share images between jobs](#using-caches-to-share-images-between-jobs)
  - [Using the cached image to build the project](#using-the-cached-image-to-build-the-project)
  - [Running the tests and storing the coverage report as an artifact](#running-the-tests-and-storing-the-coverage-report-as-an-artifact)
- [Conclusion](#conclusion)
- [References](#references)

## A "modern" development environment

Throughout this article, we'll set up a complete, containerized development environment for a `C` project:

- We'll create a _Docker_ image that we can use as a development container with [vscode](https://code.visualstudio.com/)
- Based on a minimal _Dummy_ library, we'll set up the tools to build the library within the container.
- We'll set up the static code analyzer [`clang-tidy`](https://clang.llvm.org/extra/clang-tidy/) to check our code for common mistakes.
- [`clang-format`](https://clang.llvm.org/docs/ClangFormat.html) will help us to keep our code base nicely formatted and tidy.
- We'll set up the [Unity](https://www.throwtheswitch.org/unity), executed on host via [Ceedling](https://www.throwtheswitch.org/ceedling) to test our dummy function.
- And finally, we'll set up a GitHub workflow to execute, build, and test our project using the same _Docker_ image that we're using locally.

> **Disclaimer:** There are endless possibilities to set up environments and this is just one of them. Please be lenient in case some of my solutions are not ideal or could be solved differently.

Throughout this article, I'll make use of the [Docker](https://www.docker.com/) command line interface. It is, however, out of the scope to discuss the basic concepts or the _Docker_ command line parameters. In case you don't understand why certain parameters are needed, I'd kindly ask you to refer to the online documentation.

## Containers for development

Working with embedded systems or C/C++ in general sometimes requires installing lots of specialized tools or compilers. If you are working on different projects at the same time, you'll easily end up with conflicting versions. Therefore, whenever feasible, I tend to run everything within a [Docker](https://www.docker.com/) _container_.

Another benefit is that you can also use your _Dockerfiles_ as recipes in case you need to install tools locally. The real benefit, however, is that it makes it easy for anyone to join a project by using either a prebuilt image or by attempting to build the image locally. Also, your CI can use the same environment as you do!

### Why Docker? Where's the catch?

Don't get me wrong, _Docker_ is far from being perfect and I experienced plenty of issues personally, but it is a decent solution if you keep some limitations in mind. If you're new to _Docker_, maybe consider the following:

- _Dockerfiles_ are not stable. A _Dockerfile_ that built just fine yesterday might fail to build today. There are simply too many external dependencies.
- _Docker_ is not really platform-independent. Especially if you're running a container on other CPU architectures, e.g., Apple ARM, you'll notice that some things don't run. We'll see this later.
- Some _Docker_ features are only supported in Linux or in dedicated Windows containers. E.g., mounting a USB device into a Docker container is not supported on all platforms; a limitation known [since 2016](https://github.com/docker/for-mac/issues/900).
- _Docker_ is big and thus - just like all the big companies out there - some praised as solutions may be deprecated. I personally experienced this with [docker-machine](https://github.com/docker/roadmap/issues/245). It is always best not to bet 100% on one technology and you always need to be prepared to switch.

With all this in mind, _Docker_ images are an elegant solution for development environments, especially if you have the chance to spin up your own registry. This sounds harder than it is since solutions already exist, e.g., [Google Cloud](https://cloud.google.com), [JFrog Artifactory](https://jfrog.com/artifactory/), and other providers.

I could personally experience the huge upside of this with my first dockerized environment: By pulling the images from a registry of a project that was parked for over four years, I had the development environment back up and running in under 10 minutes.

### Let's do this!

Let's start from zero with an empty repository:

```bash
$ mkdir cproject
$ cd cproject
$ git init
```

Make sure you have [Docker](https://www.docker.com/) installed and running. Create the following [`builder.Dockerfile`](builder.Dockerfile) in the root directory of the project:

```dockerfile
ARG base_tag=bullseye
ARG base_img=mcr.microsoft.com/vscode/devcontainers/base:dev-${base_tag}

FROM --platform=linux/amd64 ${base_img} AS builder-install

RUN apt-get update --fix-missing && apt-get -y upgrade
RUN apt-get install -y --no-install-recommends \
    apt-utils \
    curl \
    cmake \
    build-essential \
    gcc \
    g++-multilib \
    locales \
    make \
    ruby \
    gcovr \
    wget \
    && rm -rf /var/lib/apt/lists/*
```

> **Note:** Throughout this article, I'll only show relevant parts of the _Dockerfile_ and may omit some sections that you'll find in the [_Dockerfile_](builder.Dockerfile).

This _Dockerifle_ specifies a base image and installs some packages that we'll use in future steps. I won't go into detail about each and every package: When creating your own image you'll soon notice if something is missing and you can just extend this list of packages. What is important is the following:

- For the base image, I highly recommend using a specific _tag_. The `apt` packages vary wildly between base images, so choosing a tag buys you a bit more time until your _Dockerfiles_ fail, e.g., when the `apt` registry changes.
- For development images, I tend to use a specific **platform**. This is important in later steps, where specific tools might not exist for the CPU architecture of your machine, e.g., if you're using an Apple ARM-based computer.

If you're not using [vscode](https://code.visualstudio.com/) you could also specify a different base image, e.g., `base_img=debian:${base_tag}`. In this article we're spinning up a development environment with `vscode` and we'll therefore stick to an image that is supported by its [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers).

The image is built with the following command, which might take a little time to execute:

```bash
$ docker build -f builder.Dockerfile -t cproject-builder:latest .
```

> **Note:** Running this command again will execute much faster since all steps within a _Dockerfile_ are cached! Only if, e.g., you add new packages to the list, the whole step will be re-executed.

### A shortcut for verbose command line invocations

Docker commands are quite verbose, and they only tend to become even longer, which is why I typically park my most used commands in a [`makefile`](makefile) in the project root. Assuming you have `make` installed, you could do the following to make your life a little easier. Your future self will thank you for not having to remember all commands:

```makefile
project_name=cproject
builder-build :
	docker build -f builder.Dockerfile -t $(project_name)-builder:latest .
```

Now, all you need to do is the following to rebuild your image:

```bash
$ make builder-build
```

> **Note:** A more elegant solution than `makefiles` may be [Invoke](http://www.pyinvoke.org/), presented in an [article on the interrupt blog](https://interrupt.memfault.com/blog/building-a-cli-for-firmware-projects). Since I'm a dinosaur, however, and for the sake of simplicity, I'll stick to `makefiles` in this article.

Let's spin up a container from our image and take it for a test drive:


```bash
$ docker run --rm -it --platform linux/amd64 cproject-builder:latest /bin/bash
$
$ gcc --version
gcc (Debian 10.2.1-6) 10.2.1 20210110
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
$
$ cmake --version
cmake version 3.18.4

CMake suite maintained and supported by Kitware (kitware.com/cmake).
$ exit
```

When using `apt`, the version of the tools that have been installed depends on the base image and therefore on the package registry. If you want to install specific versions, you can do this by executing custom `RUN` commands as we'll see later.



## Visual Studio Code Dev Containers

The downside of **not** having all tools installed locally is that those tools can not be leveraged by your IDE of choice. E.g., when using `vscode` you won't be able to properly set up intellisense or any other helpers if you don't have any compiler installed.

`vscode` allows you to run an instance of the editor within a so-called _development container_. This is also the reason why we chose the `mcr.microsoft.com/vscode/devcontainers/base` image as the base image: We can connect `vscode` within our builder container and therefore have all the tools available! Have a look at the [tutorial](https://code.visualstudio.com/docs/devcontainers/tutorial) for the exact steps or in case anything fails whilst following along this article. For now, I assume that you have `vscode` and the [Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) extension installed.

By creating the [`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json) file, we can tell `vscode` to use our newly built image for its development container, and we'll install some extensions in the container along the way:

```json
{
  "name": "C",
  "build": {
    "dockerfile": "../builder.Dockerfile"
  },
  "runArgs": ["--platform=linux/amd64"],
  "customizations": {
    "vscode": {
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash",
        "extensions.verifySignature": false
      },
      "extensions": ["ms-vscode.cpptools", "ms-vscode.cmake-tools", "cschlosser.doxdocgen"]
    }
  }
}
```

If you now reload your window or reopen `vscode` (using the `cproject` folder as the root folder!) `vscode` should now ask you if it should use the detected development container.

<img src="assets/vscode-reload.png?raw=true">

It'll take a while for `vscode` to set up your container and install the extensions, but in the end you should have `vscode` connected to your Linux container, with all previous tools installed.

<img src="assets/vscode-remote.png?raw=true">

## A skeleton subsystem

For this article, we'll create a single `.c` and `.h` pair, built using [cmake](https://cmake.org/), with a single random number generator function, as follows:

```bash
$ tree --charset=utf-8 --dirsfirst
.
├── include
│   └── dummy
│       └── dummy.h
├── src
│   └── dummy.c
├── CMakeLists.txt
├── builder.Dockerfile
└── makefile <- this is just our makefile containing the docker commands
```

```c
#include "dummy/dummy.h"
uint8_t dummy_random(void)
{
    return 4U; // determined by fair dice roll.
}
```

I'll spare you the details, the `CMakeLists.txt` simply defines a library named "Dummy" and adds the corresponding files to the library. Have a look at the [`CMakeLists.txt`](CMakeLists.txt) file. What's important is: This already builds within our development container and `vscode`! Open the integrated terminal in your remote instance and execute `cmake` as follows.

```bash
$ cmake -B build
-- The C compiler identification is GNU 10.2.1
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /workspaces/cproject/build
$ cmake --build build
Scanning dependencies of target Dummy
[ 50%] Building C object CMakeFiles/Dummy.dir/src/dummy.c.o
[100%] Linking C static library libDummy.a
[100%] Built target Dummy
```



## Building outside of `vscode` and mounting volumes

Not everyone is a friend of `vscode`. All the magic happening behind the scenes in an IDE is not everyone's favorite flavor and it is always good to know how to handle things without an IDE, so I'd like to provide an alternative setup that will also be important once we deal with the GitHub workflow, where we simply have to step out of the comforts of an IDE.

One such magic step that `vscode` does for you - and that we'll now need to do ourselves - is managing the files within the root container. We'll use a [volume](https://docs.docker.com/storage/volumes) to make all files available within the container. At the same time this makes all modifications that happen _inside_ the container are also visible _outside_ of it. We update our _Dockerfile_ with the following commands:

```dockerfile
VOLUME ["/builder/mnt"]
WORKDIR /builder/mnt
```

This defines a volume that we can later use to mount our project directory when executing `docker run`, and the `workdir` instruction tells _Docker_ to make this the default path for all future steps. Don't forget to rebuild the image, otherwise, the changes will not be available!

```bash
$ make builder-build
```

### A note on Docker volumes

As mentioned in the introduction, _Docker_ does not behave the same way on all platforms. Only on Linux or when using Windows containers on Windows, containers run "natively" and thus without major penalties. If you're running a Linux container on macOS or Windows - in very simple words - they execute within a VM (though admittedly it is a bit more complicated than that).

The key takeaway is the following: Since the container is executed in a VM, the I/O performance is significantly worse compared to a container that is run natively. For compiled languages or for any process that creates a lot of files, this impact can be significant since the overhead can be up to 100x of what you're experiencing natively. This can lead to longer build or generation times.

Thankfully, _Docker_ has provided a good solution for macOS with the `VirtioFS` file-sharing implementation. All you need to do is enable it in your _Docker_ configuration:

<img src="assets/docker-sharing.png?raw=true">

For Windows, at the time of writing, I don't know of a similar solution so you might have to dig a little deeper in case you run into performance problems. Personally, I've successfully used [docker-sync](https://docker-sync.io/) before, but it is a bit harder to handle.

### Running the development container manually

Now we can run the following command to spin up the container with our project folder mounted as a volume:

```bash
$ docker run \
    --rm \
    -it \
    --platform linux/amd64 \
    --workdir /builder/mnt \
    -v .:/builder/mnt \
    cproject-builder:latest \
    /bin/bash
```

The current directory "`.`" is now available within the container as `/builder/mnt`. And - just like in our development container - we're able to build the project:

```bash
$ rm -rf build
$ cmake -B build
-- The C compiler identification is GNU 10.2.1
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /builder/mnt/build
$ cmake --build buildcmake --build build
Scanning dependencies of target Dummy
[ 50%] Building C object CMakeFiles/Dummy.dir/src/dummy.c.o
[100%] Linking C static library libDummy.a
[100%] Built target Dummy
```

Since the command to spin up the container is rather verbose, I also tend to also use a `make` target in my `makefile` for that, such that all I need to type is `make builder-run`:

```makefile
builder-run :
	docker run \
		--rm \
		-it \
		--platform linux/amd64 \
		--workdir /builder/mnt \
		-v .:/builder/mnt \
		$(project_name)-builder:latest \
		/bin/bash
```



## Installing `clang` tools for formatting and static code analysis

The flexibility of C and C++ comes with plenty of massive footguns so, personally, I try to add at least a minimal static code analysis to my project. This helps to catch the most obvious mistakes and gives some extra confidence in the codebase. There are plenty of tools out there, but my personal preference so far is [`clang-tidy`](https://clang.llvm.org/extra/clang-tidy/).

Also, when collaborating on a codebase, a formatter is always a nice thing to have, and since we're installing `clang-tidy` anyhow, we might as well go ahead and install [`clang-format`](https://clang.llvm.org/docs/ClangFormat.html).

The two configuration files [`.clang-format`](.clang-format) and [`.clang-tidy`](.clang-tidy) are placed into the root directory of the project such that any IDE picks them up automatically. I won't go into detail about the options of the two tools and will just assume that the reader follows the previous links. So this is what we have now:

```bash
$ tree --charset=utf-8 --dirsfirst -a -I build -I .git -L 1
.
├── .devcontainer
├── include
├── src
├── .clang-format
├── .clang-tidy
├── CMakeLists.txt
├── builder.Dockerfile
└── makefile
```

### Installing specific in the builder image

Here's a catch with `clang-tidy` and `clang-format`: The configuration files must match the tool version, they are not necessarily backward compatible. It is therefore important to install compatible versions for your configuration files. Since we have a container we'll always be using the correct version! The installation, however, can be tricky. Why? Running `apt-get` to install the tools does not always work since the package registries typically contain ancient tool versions. One way around this is to install them manually in our _Dockerfile_:

```dockerfile
ARG base_tag=bullseye
ARG llvm_version=16
RUN apt-get update --fix-missing && apt-get -y upgrade
RUN apt-get install -y --no-install-recommends \
    gnupg2 \
    gnupg-agent \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN curl --fail --silent --show-error --location https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -
RUN echo "deb http://apt.llvm.org/$base_tag/ llvm-toolchain-$base_tag-$llvm_version main" >> /etc/apt/sources.list.d/llvm.list

RUN apt-get update --fix-missing && apt-get -y upgrade
RUN apt-get install -y --no-install-recommends \
    clang-format-${llvm_version} \
    clang-tidy-${llvm_version} \
    && rm -rf /var/lib/apt/lists/*

RUN ln -s /usr/bin/clang-format-${llvm_version} /usr/local/bin/clang-format
RUN ln -s /usr/bin/clang-tidy-${llvm_version} /usr/local/bin/clang-tidy
```

This is one of the steps in a _Dockerfile_ that I've previously mentioned which can break quite easily in case you're trying to rebuild an image in a couple of years. In case you're happy with the packages provided for your base images, you can also simply install the packages that are provided with the `apt` registry. Now both `clang-format` and `clang-tidy` are available in the image:

```bash
$ make builder-build
$ make builder-run

$ clang-format --version
Debian clang-format version 16.0.6 (++20230704084212+7cbf1a259152-1~exp1~20230704204302.101)

$ clang-tidy --version
Debian LLVM version 16.0.6
  Optimized build.
```

### Using `clang-format` in the Development Container

`vscode` typically needs to be instructed to rebuild the development container from the _Dockerfile_. You can do so explicitly using the command prompt:

<img src="assets/vscode-rebuild.png?raw=true">

Now you have `clang-tidy` and `clang-format` available within your development container. You can now also instruct `vscode` to automatically format your files on save. For this, we create a workspace configuration [`.vscode/settings.json`](.vscode/settings.json) file:

```json
{
  "[c]": {
    "editor.formatOnSave": true
  },
  "[cpp]": {
    "editor.formatOnSave": true
  },
  "C_Cpp.default.compileCommands": "${workspaceFolder}/src/build/compile_commands.json",
  "cmake.configureOnOpen": true
}
```

I've hidden a little detail here: `clang-tidy` operates on the `compile_commands.json` file that can be created when using `cmake` as a build system. I've enabled this option within the [`CMakeLists.txt`](CMakeLists.txt) file. At the same time, we're allowing the `cmake` extension for `vscode` to configure the project when opening and therefore have all the fancy buttons and actions available to build the project in case you don't like the command line. Personally, I still fall back to the command line. But then, I'm a dinosaur.

Now, if you open the `dummy.c` file in your editor it will automatically format it according to the rules in your `.clang-format` file whenever you save the file. But what if someone in your team doesn't use `vscode` or forgot to enable the format-on-save feature?

### Wrapping `clang-format` and `clang-tidy` calls

This section is a little self-promotion of two wrapper scripts for `clang-format` and `clang-tidy` that I am maintaining: [run-clang-tidy](https://github.com/lmapii/run-clang-tidy) and [run-clang-format](https://github.com/lmapii/run-clang-format). Both tools are command line tools written in [Rust](https://www.rust-lang.org/) that simplify and parallelize the execution of `clang-format` and `clang-tidy` from the command line.

Installing the tools within your container _should_ be easy via the [Rust package manager `cargo`](https://github.com/rust-lang/cargo), however, at the time of writing, I could not install any binaries inside a container via the `cargo` `apt` package, and installing the tools from scratch within the container simply took too much time, so I decided to download the released, architecture-specific version instead:

```dockerfile
RUN mkdir -p /usr/local/run-clang-format
RUN wget -O clang-utils.tgz "https://github.com/lmapii/run-clang-format/releases/download/v1.4.10/run-clang-format-v1.4.10-i686-unknown-linux-gnu.tar.gz" && \
    tar -C /usr/local/run-clang-format -xzf clang-utils.tgz --strip-components 1 && \
    rm clang-utils.tgz
ENV PATH /usr/local/run-clang-format:$PATH
RUN run-clang-format --version

RUN mkdir -p /usr/local/run-clang-tidy
RUN wget -O clang-utils.tgz "https://github.com/lmapii/run-clang-tidy/releases/download/v0.2.1/run-clang-tidy-v0.2.1-i686-unknown-linux-gnu.tar.gz" && \
    tar -C /usr/local/run-clang-tidy -xzf clang-utils.tgz --strip-components 1 && \
    rm clang-utils.tgz
ENV PATH /usr/local/run-clang-tidy:$PATH
RUN run-clang-format --version
```

These wrapper scripts allow to define the execution of `clang-format` and `clang-tidy` based on `.json` input files. Within these files, it is possible to efficiently filter all files, e.g., third-party software for which the format-on-save would be more cumbersome than helpful. Since we'll be adding unit tests later, we can prepare the configurations already:

For formatting, we simply need to provide the paths to the source files and filters for files that we want to exclude from formatting or from the glob expansion:

```json
{
  "paths": ["./src/**/*.[ch]", "./include/**/*.[h]", "./tests/unittest/test/*.[ch]"],
  "filterPre": [".*"],
  "filterPost": ["./src/build/**", "./tests/unittest/build/**", "./tests/unittest/generated"]
}
```
For the static analysis, we also need to tell the tool where to find the `compile_commands.json`:

```json
{
  "paths": ["./src/**/*.[ch]", "./include/**/*.[h]"],
  "filterPre": [".*"],
  "filterPost": ["./build/**"],
  "buildRoot": "./build"
}
```

Now we just need to update our image and run the commands. Whether you do this by spinning up your own container or by rebuilding the `vscode` development container is up to you, it should run in both environments:

```bash
$ make builder-build
$ make builder-run

$ run-clang-format clang-format.json
 [ 1/5 ] No style file specified, assuming .clang-format exists in the project tree
 [ 2/5 ] Found 2 files for the provided path patterns
 [ 3/5 ] Found clang-format version 16.0.6 using command clang-format
 [ 4/5 ] Executing clang-format ...

  Formatting /builder/mnt/src/dummy.c
  Formatting /builder/mnt/include/dummy/dummy.h
     Running [==========================] 2/2
     Finished in 0 seconds

$ run-clang-tidy clang-tidy.json

 [ 1/6 ] No tidy file specified, assuming .clang-tidy exists in the project tree
 [ 2/6 ] Using build root /workspaces/cproject/build
 [ 3/6 ] Found 1 files for the provided path patterns
 [ 4/6 ] Found clang-tidy version 16.0.6 using command clang-tidy
 [ 5/6 ] Executing clang-tidy ...

          Ok /workspaces/cproject/src/dummy.c
     Running [==========================] 1/1
     Finished in 1 second
```

> **Note:** Instead of running `cmake --build build` every time you modify your files, you can now also simply execute the static code analysis instead. It also compiles your files, but at the same time points out some of the mistakes that you have enabled.



## Adding unit tests

We're already able to build our library, analyze it and have a formatter in place. One more thing that we need in our development environment is a unit test framework.

For this article, I've chosen the [Unity](https://www.throwtheswitch.org/unity) test framework, executed on the host via [Ceedling](https://www.throwtheswitch.org/ceedling). I'm using this framework since it is quite easy to extend the integration to also execute tests on an actual target hardware in case you're developing an embedded system.

### Installing `Unity` and `Ceedling` in the builder image

In the first step of our builder image we already installed `ruby`, so installing our unit test tools is very straightforward:

```dockerfile
# install unity cmock and ceedling (unit test environment)
RUN gem install ceedling
# set standard encoding to UTF-8 for ruby (and thus ceedling)
ENV RUBYOPT "-KU -E utf-8:utf-8"
```

After rebuilding our image we're good to go!

```bash
$ make builder-build
```

### Configuring `Unity` and running unit tests

Lots of resources exist on how to set up `Unity` and `Ceedling`, including an [excellent article on Embedded Artistry](https://embeddedartistry.com/blog/2019/2/25/unit-testing-and-reporting-on-a-build-server-using-ceedling-and-unity/) or the [_ThrowTheSwitch's_ own articles](https://www.throwtheswitch.org/articles). In short, you set the configuration switches for `Unity` in a dedicated [`unity_config.h`](tests/unittest/support/unity_config.h) file, and configure `Ceedling` with the [`project.yml`](tests/unittest/project.yml). `Ceedling` is a neat little helper that generates all the test runners for you, so all you need to do is add your own test files and tell `Ceedling` how to detect them. Have a look at the files in this repository in case you need details!

In the [`project.yml`](tests/unittest/project.yml) we told `Ceedling` that our tests will be located in the `tests/unittest/test` directory and all test file names have the prefix `test`. So all we need to do is to add our [`test_dummy.c`](tests/unittest/test/test_dummy.c) file to test our implementation, leading to the following files added:


```bash
$ tree --charset=utf-8 --dirsfirst -- tests
tests
└── unittest
    ├── support
    │   └── unity_config.h
    ├── test
    │   └── test_dummy.c
    └── project.yml
```

We can then create our first unit test `tests/unittest/test/test_dummy.c`:

```c
#include "dummy/dummy.h"
#include "unity.h"

void setUp(void) {}
void tearDown(void) {}

void test_dummy(void)
{
    TEST_ASSERT_EQUAL(4U, dummy_random());
}
```

If you're in your `vscode` development container, you can run the following straight from the integrated terminal. Otherwise, spin up the container using `make builder-run` to execute your tests:

```bash
$ ceedling test:all

Test 'test_dummy.c'
-------------------
Generating runner for test_dummy.c...
Compiling test_dummy_runner.c...
Compiling test_dummy.c...
Compiling unity.c...
Compiling dummy.c...
Compiling cmock.c...
Linking test_dummy.out...
Running test_dummy.out...

--------------------
OVERALL TEST SUMMARY
--------------------
TESTED:  1
PASSED:  1
FAILED:  0
IGNORED: 0
```

### Bonus: Coverage reports

The previously mentioned [article on Embedded Artistry](https://embeddedartistry.com/blog/2019/2/25/unit-testing-and-reporting-on-a-build-server-using-ceedling-and-unity/) also shows how to configure coverage reports. This is already set up in the [`project.yml`](tests/unittest/project.yml), and since we included `gcovr` in the `apt` packages of our image, we are good to run our unit tests with coverage:

```bash
$ ceedling gcov:all

Test 'test_dummy.c'
-------------------
Compiling test_dummy_runner.c...
Compiling test_dummy.c...
Compiling dummy.c with coverage...
Compiling unity.c...
Compiling cmock.c...
Linking test_dummy.out...
Running test_dummy.out...

--------------------------
GCOV: OVERALL TEST SUMMARY
--------------------------
TESTED:  1
PASSED:  1
FAILED:  0
IGNORED: 0

---------------------------
GCOV: CODE COVERAGE SUMMARY
---------------------------
dummy.c Lines executed:100.00% of 2
dummy.c No branches
dummy.c No calls
```

The [`project.yml`](tests/unittest/project.yml) also sets up a utility for generating coverage reports. In this demo project we can generate a `gcovr` HTML report with the following command:

```bash
$ ceedling utils:gcov
Creating gcov results report(s) in 'build/artifacts/gcov'... Done in 1.359 seconds.
```

You'll now find the HTML report in `tests/unittest/build/artifacts/gcov/GcovCoverageResults.html`.

<img src="assets/gcovr-report.png?raw=true">



## Getting started with GitHub workflows

We got quite far already, and this setup is way beyond the "works on my machine" type of environment: All of your contributors can build a development container and execute all steps to build or test your project. But there is one more thing that we should do to keep the project stable even if we're not actively working on it: Setting up a pipeline that periodically ensures that the environment still works.

Continuously testing and checking your code is important. Pipelines are now available on all major platforms such as [GitHub](https://github.com/), [GitLab](https://about.gitlab.com/), or [Bitbucket](https://bitbucket.org/). In this article we'll set up a [GitHub action](https://github.com/features/actions) which periodically builds our image and tests our code.

There's yet another catch though: Ensuring that your Docker images build and using those Docker images in your pipeline can sometimes be a bit tedious. At the time of writing, GitHub has support for so-called [container actions](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action) which allows executing in pipeline steps using a Docker container. These steps, however, assume that the image exists on some registry, e.g., [dockerhub](https://hub.docker.com/), and is not built within the action itself.

Since nowadays [dockerhub](https://hub.docker.com/) is no longer entirely free to use, we will still try to execute all steps within a Docker image that is built as part of the pipeline. This makes the pipeline steps less elegant and very verbose, but I'll choose this approach over having to use multiple actions or a container registry.

### Adding a GitHub action

GitHub actions are added `.yml` workflow files in the `.github/workflows` directory of the repository. We'll name our workflow file [`ci.yml`](.github/workflows/ci.yml):

```yml
name: ci
on:
  push:
    branches:
    - main
  schedule:
    - cron: '0 1 * * 0'
jobs:
  # empty
```

With the above configuration, the jobs in this action are executed [at 01:00 on every Sunday](https://crontab.guru/#0_1_*_*_0) and on every push to the `main` branch.

### Building an image in a GitHub action

Building a Docker image in your pipeline is quite straightforward due to the existing GitHub actions:

```yml
jobs:
  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
        name: Build builder-image
        with:
          context: .
          file: builder.Dockerfile
          tags: cproject-builder:latest
  build-and-test:
    needs: docker-build
```

The problem is that all jobs are isolated and that data _including Docker images_ must be shared explicitly between jobs. Without sharing, `cproject-builder:latest` is not available in the next job `build-and-test`.

### Using caches to share images between jobs

The [proposed solution by Docker](https://docs.docker.com/build/ci/github-actions/share-image-jobs/) is to share an image using _artifacts_. Artifacts are files that can be stored in steps in a GitHub action and thus be downloaded in a subsequent step. For our builder image this is very problematic since the builder image is very large:

- Uploading the image takes a very long time.
- Artifact count towards the storage limit in your GitHub plan. A free plan only has 500 MB of storage.
- Downloading the image in subsequent steps also takes a very long time.

My favorite solution for this problem is using [cache dependencies](https://github.com/actions/cache). Similar to the proposed solution by Docker, we build the image and store it in a file, which is then accessible using a _cache_. Another benefit of this approach is, that for subsequent actions and if the _Dockerfile_ did not change, the image is not rebuilt!

```yml
jobs:
  docker-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
      -
        name: Create docker cache folder
        run: mkdir -p /tmp/docker
      -
        name: Restore docker image
        id: cache-docker
        uses: actions/cache@v3
        with:
          path: /tmp/docker
          key: ${{ runner.os }}-docker-${{ hashFiles('builder.Dockerfile') }}
      -
        name: Build docker builder-image
        if: steps.cache-docker.outputs.cache-hit != 'true'
        uses: docker/build-push-action@v4
        with:
          context: .
          file: builder.Dockerfile
          tags: cproject-builder:latest
          outputs: type=docker,dest=/tmp/docker/${{ runner.os }}-builder-image.tar
```

At the time of writing caches do **not** count towards the storage limit of your GitHub account. The only downside that I've encountered, is that caches are not shared between actions of different branches and that you'll need to manage your caches in case you have too many branches using images.

### Using the cached image to build the project

In the next job, we load the builder image and execute our actions. As mentioned before, the image is not available yet and therefore we cannot use the much more elegant GitHub actions for containers. It does, however, do its job and does so efficiently since restoring a cache takes much less time than downloading an artifact:

```yml
jobs:
  docker-build:
    # <snip>
  build-and-test:
    runs-on: ubuntu-latest
    needs: docker-build
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
      -
        name: Restore docker image
        id: cache-docker
        uses: actions/cache@v3
        with:
          path: /tmp/docker
          key: ${{ runner.os }}-docker-${{ hashFiles('builder.Dockerfile') }}
      -
        name: Load image
        run: |
          docker load --input /tmp/docker/${{ runner.os }}-builder-image.tar
          docker image ls -a
      -
        name: Build library
        run: |
          docker run \
            --rm \
            --platform linux/amd64 \
            --workdir /builder/mnt \
            -v ${{ github.workspace }}:/builder/mnt \
            cproject-builder:latest \
            /bin/bash -c "rm -rf build; cmake -B build; cmake --build build"
```

This job needs the previous `docker-build` to pass, then restores the docker image from the cache and builds the project within the image using `cmake`. Here the downside of this approach is visible: We always need to use the entire `docker run` command to execute our steps.

### Running the tests and storing the coverage report as an artifact

We can now run the same commands that we used previously to run the unit tests with coverage and generate our coverage report. In addition, we can store this coverage report as an artifact in our pipeline such that we can download it after any successful run.

```yml
jobs:
  docker-build:
    # <snip>
  build-and-test:
    runs-on: ubuntu-latest
    needs: docker-build
    steps:
      # <snip>
      -
        name: Test library
        run: |
          docker run \
            --rm \
            --platform linux/amd64 \
            --workdir /builder/mnt/tests/unittest \
            -v ${{ github.workspace }}:/builder/mnt \
            cproject-builder:latest \
            /bin/bash -c "ceedling clobber; ceedling gcov:all; ceedling utils:gcov"
      -
        name: Archive coverage results
        shell: bash
        run: |
          staging="reports-${{github.run_number}}"
          mkdir -p "$staging"
          cp -r tests/unittest/build/artifacts/gcov "$staging"
          tar czf "$staging.tar.gz" "$staging"
          echo "ASSET=$staging.tar.gz" >> $GITHUB_ENV
      -
        name: Archive artifacts
        uses: actions/upload-artifact@v3
        with:
          name: reports-${{github.run_number}}
          path: ${{ env.ASSET }}
          retention-days: 3
```

Never forget to specify a retention period for your artifacts, since otherwise, it is very easy to hit the storage limit of your GitHub plan.

## Conclusion

If you've managed to bear with me until this point, thank you for reading! Our job is done here, the library builds, the report is available as an artifact, and all of this without clogging our computer with tools.

<img src="assets/github-action.png?raw=true">

But are we really ever done? A development environment grows with the project, and so will your base image. Even at this point it grew fairly large, and if that is a problem for you or your contributors, you should look into some of the [strategies to reduce your image size](https://www.docker.com/blog/reduce-your-image-size-with-the-dive-in-docker-extension/). Another strategy is to split images based on their usage, e.g., use a different image for building and another one for testing. This can greatly reduce your image size and allows you to use already stripped down base images. This discussion, however, is far beyond the scope of this article.

You have now all the skills to set up a containerized C/C++ project with a CI on GitHub. With this at hand, it should be easy to add a specific compiler to your _Dockerfile_ or to clean up all the mistakes that I did when setting up the CI.

## References

- [Visual Studio Code Dev Containers tutorial](https://code.visualstudio.com/docs/devcontainers/tutorial)
- [The Unity test framework](https://www.throwtheswitch.org/unity)
- [Embedded Artistry's article on Ceedling and Unity on a build server](https://embeddedartistry.com/blog/2019/2/25/unit-testing-and-reporting-on-a-build-server-using-ceedling-and-unity/)
- [GitHub actions documentation](https://github.com/features/actions)
