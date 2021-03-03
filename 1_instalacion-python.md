<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_1

-->
# Installing Python

Before we start using Python, we need to install it. Some operating systems, like Mac OS X and some Linux variants, have Python preinstalled. Those versions of Python, colloquially called “system Python,” often make poor defaults for people who want to develop in Python.

For one thing, the version of Python installed is often behind latest practices. For another, system integrators will often patch Python in ways that can lead to surprises later. For example, Debian-based Python is often missing modules like venv and ensurepip. Mac OS X Python links against a Mac shim around its native SSL library. What those things means, especially when starting out and consuming FAQs and web resources, is that it is better to install Python from scratch.

We cover a few ways to do so and the pros and cons of each.

## 1.1 OS Packages

For some of the more popular operating systems, volunteers have built ready-to-install packages.

The most famous of these is the “deadsnakes” PPA (Personal Package Archives). The “dead” in the name refers to the fact that those packages are already built – with the metaphor that sources are “alive.” Those packages are built for Ubuntu and will usually support all the versions of Ubuntu that are still supported upstream. Getting those packages is done simply:

```
$ sudo add-apt-repository ppa:deadsnakes/ppa
$ sudo apt update
```

On Mac OS, the homebrew third-party package manager will have Python packages that are up to date. An introduction to Homebrew is beyond the scope of this book. Since Homebrew is a rolling release, the version of Python will get upgraded from time to time. While this means that it is a useful way to get an up-to-date Python, it makes for a poor target for reliably distributing tools.

It is also a choice with some downsides for doing day-to-day development. Since it upgrades quickly after a new Python releases, this means development environments can break quickly and without warning. It also means that sometimes code can stop working: even if you are careful to watch upcoming Pythons for breaking changes, not all packages will. Homebrew Python is a good fit when needing a well-built, up-to-date Python interpreter for a one-off task. Writing a quick script to analyze data, or automate some APIs, is a good use of Homebrew Python.

Finally, for Windows, it is possible to download an installer from Python.org for any version of Python.

## 1.2 Using Pyenv

Pyenv tend to be the highest Return on Investment for installing Python for local development. The initial setup does have some subtleties. However, it allows installing as many Python versions side by side as are needed. It allows managing how one will be accessed: on either per-user default or a per-directory default.

Installing pyenv itself depends on the operating system. On a Mac OS X, the easiest way is to install it via Homebrew. Note that in this case, pyenv itself might need to be upgraded to install new versions of Python.

On a UNIX-based operating system, such as Linux or FreeBSD, the easiest way to install pyenv is by using the curl|bash command:

```
$ PROJECT=https://github.com/pyenv/pyenv-installer \
  PATH=raw/master/bin/pyenv-installer \
  curl -L $PROJECT/PATH | bash
```

Of course, this comes with its own security issues and could be replaced with a two-step process:

```
$ git clone https://github.com/pyenv/pyenv-installer
$ cd pyenv-installer
$ bash pyenv-installer
```

where one can inspect the shell script before running, or even use git checkout to pin to a specific revision.

Sadly, pyenv does not work on Windows.

After installing pyenv, it is useful to integrate it with the running shell. We do this by adding to the shell initialization file (for example, .bash_profile):

```
export PATH="~/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

This will allow pyenv to properly intercept all needed commands.

Pyenv separates the notion of installed interpreters from available interpreters. In order to install a version,

```
$ pyenv install <version>
```

For CPython, <version> is just the version number, such as 3.6.6 or 3.7.0rc1.

An installed version is distinct from an available version. Versions can be available either “globally” (for a user) by using

```
$ pyenv global 3.7.0
```

or locally by using

```
$ pyenv local 3.7.0
```

Local means they will be available in a given directory. This is done by putting a file python-version.txt in this directory. This is important for version-controlled repositories, but there are a few different strategies to manage those. One is to add this file to the “ignored” list. This is useful for heterogenous teams of open source projects. Another is to check this file in, so that the same version of Python is used in this repository.

Note that pyenv, since it is designed to install versions of Python side by side, has no concept of “upgrading” Python. In order to use a newer Python, it needs to be installed with pyenv and then set as the default.

By default, pyenv installs non-optimized versions of Python. If optimized versions are needed,

```
env PYTHON_CONFIGURE_OPTS="--enable-shared
                           --enable-optimizations
                           --with-computed-gotos
                           --with-lto
                           --enable-ipv6" pyenv install
```

will build a version that is pretty similar to binary versions from python.org.

## 1.3 Building Python from Source

The main challenge in building Python from source is that, in some sense, it is too forgiving. It is all too easy to build it with one of the built-in modules disabled because its dependency was not detected. This is why it is important to know what dependencies are that fragile, and how to make sure a local installation is good.

The first fragile dependency is ssl. It is disabled by default and must be enabled in Modules/Setup.dist. Carefully follow the instructions there about the location of the OpenSSL library. If you have installed OpenSSL via system packages, it will usually be in /usr/. If you have installed it from source, it will usually be in /usr/local.

The most important thing is to know how to test for it. When Python is done building, run ./python.exe -c 'import _ssl'. That .exe is not a mistake – this is how the build process calls the just-built executable, which is renamed to python during installation. If this succeeds, the ssl module was built correctly.

Another extension that can fail to build is sqlite . Since it is a built-in, a lot of third-party packages depend on it, even if you are not using it yourself. This means a Python installation without the sqlite module is pretty broken. Test by running ./python.exe -c 'import sqlite3'.

In a Debian-based system (such as Ubuntu), libsqlite3-dev is required for this to succeed. In a Red Hat-based system (such as Fedora or CentOS), libsqlite3-dev is required for this to succeed.

Next, check for _ctypes with ./python.exe -c 'import _ctypes'. If this fails, it is likely that the libffi headers are not installed.

Finally, remember to run the built-in regression test suite after building from source. This is there to ensure that there have been no silly mistakes while building the package.

## 1.4 PyPy

The “usual” implementation of Python is sometimes known as “CPython,” to distinguish it from the language proper. The most popular alternative implementation is PyPy. PyPy is a Python-based JIT implementation of Python in Python. Because it has a dynamic JIT (Just in Time) compilation to assembly, it can sometimes achieve phenomenal speed-ups (3x or even 10x) over regular Python.

There are sometimes challenges in using PyPy. Many tools and packages are tested only with CPython. However, sometimes spending the effort to check if PyPy is compatible with the environment is worth it if performance matters.

There are a few subtleties in installing Python from source. While it is theoretically possible to “translate” using CPython, in practice the optimizations in PyPy mean that translating using PyPy works on more reasonable machines. Even when installing from source, it is better to first install a binary version to bootstrap.

The bootstrapping version should be PyPy, not PyPy3. PyPy is written in the Python 2 dialect. It is one of the only cases where worrying about the deprecation is not relevant, since PyPy is a Python 2 dialect interpreter. PyPy3 is the Python 3 dialect implementation, which is usually better to use in production as most packages are slowly dropping support for Python 2.

The latest PyPy3 supports 3.5 features of Python, as well as f-strings. However, the latest async features, added in Python 3.6, do not work.

## 1.5 Anaconda

The closest to a “system Python” that is still reasonable for use as a development platform is the Anaconda Python . Anaconda is a so-called “meta-distribution.” It is, in essence, an operating system on top of the operating system. Anaconda has its grounding in the scientific computing community, and so its Python comes with easy-to-install modules for many scientific applications. Many of these modules are nontrivial to install from PyPI, requiring a complicated build environment.

It is possible to install multiple Anaconda environments on the same machine. This is handy when needing different Python versions or different versions of PyPI modules.

In order to bootstrap Anaconda, we can use the bash installer, available from https://conda.io/miniconda.html . The installer will also modify ~/.bash_profile to add the path to conda, the installer.

Conda environments are created using conda create --name <name>, and activated using source conda activate <name>. There is no easy way to use unactivated environments. It is possible to create a conda environment while installing packages in it: conda create --name some-name python. We can specify the version using = – conda create --name some-name python=3.5. It is also possible to install more packages into a conda environment, using conda install package[=version], after the environment has been activated. Conda has a lot of pre-built Python packages, especially ones that are nontrivial to build locally. This makes it a good choice if those packages are important to your use case .
