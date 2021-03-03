<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_2
-->

# Empaquetado

Much of dealing with Python in the real world is dealing with third-party packages. For a long time, the situation was not good. Things have improved dramatically, however. It is important to understand which “best practices” are antiquated rituals, which ones are based on faulty assumptions but have some merit, and which are actually good ideas.

When dealing with packaging, there are two ways to interact. One is to be a “consumer,” wanting to use the functionality from a package. Another is to be the “producer,” publishing a package. These describe, usually, different development tasks, not different people.

It is important to have a solid understanding of the “consumer” side of packages before moving to “producing.” If the goal of a package publisher is to be useful to the package user, it is crucial to imagine the “last mile” before starting to write a single line of code.

## 2.1 Pip

The basic packaging tool for Python is pip . By default, installations of Python do not come with pip. This allows pip to move faster than core Python – and work with alternative Python implementations, like PyPy. However, they do come with the useful ensurepip module. This allows getting pip via python -m ensurepip. This is usually the easiest way to bootstrap pip.

Some Python installations, especially system ones, disable ensurepip. When lacking ensurepip, there is a way of manually getting it: get-pip.py. This is a downloadable single file that, when executed, will unpack pip.

Luckily, pip is the only package that needs these weird gyrations to install. All other packages can, and should, be installed using pip. This includes upgrading pip itself, which can be done with pip install --upgrade pip.

Depending on how Python was installed, its “real environment” might or might not be modifiable by our user. Many instructions in various README files and blogs might encourage doing sudo pip install. This is almost always the wrong thing to do: it will install the packages in the global environment.

It is almost always better to install in virtual environments – those will be covered later. As a temporary measure, perhaps to install things needed to create a virtual environment, we can install to our user area. This is done with pip install --user.

The pip install command will download and install all dependencies. However, it can fail to downgrade incompatible packages. It is always possible to install explicit versions: pip install package-name==<version> will install this precise version. This is also a good way to get explicitly non-general-availability packages, such as release candidates, beta, or similar, for local testing.

If wheel is installed, pip will build, and usually cache, wheels for packages. This is especially useful when dealing with a high virtual environment churn, since installing a cached wheel is a fast operation. This is also especially useful when dealing with so-called “native,” or “binary,” packages – those that need to be compiled with a C compiler. A wheel cache will eliminate the need to build it again.

pip does allow uninstalling, with pip uninstall. This command, by default, requires manual confirmation. Except for exotic circumstances, this command is not used. If an unintended package has snuck in, the usual response is to destroy the environment and rebuild it. For similar reasons, pip install --ugprade is not often needed: the common response is to destroy and re-create the environment. There is one situation where it is a good idea: pip install --upgrade pip. This is the best way to get a new version of pip with bug fixes and new features.

pip install supports a “requirements file,” pip install --requirements or pip install -r. A requirements file simply has one package per line. This is no different than specifying packages on the command line. However, requirements files often specify “strict dependencies.” A requirements file can be generated from an environment with pip freeze. The usual way to get a requirements file is, in a virtual environment, to do the following:

```
$ pip install -e .
$ pip freeze > requirements.txt
```

This means the requirements file will have the current package, and all of its recursive dependencies, with strict versions.

## 2.2 Virtual Environments

Virtual environments are often misunderstood, because the concept of “environments” is not clear. A Python environment refers to the root of the Python installation. The reason it is important is because of the subdirectory lib/site-packages. The lib/site-packages directory is where third-party packages are installed. In modern times, they are often installed by pip. While there used to be other tools to do it, even bootstrapping pip and virtualenv can be done with pip, let alone day-to-day package management.

The only common alternative to pip is system packages, where a system Python is concerned. In the case of an Anaconda environment, some packages might be installed as part of Anaconda. In fact, this is one of the big benefits of Anaconda: many Python packages are custom built, especially those which are nontrivial to build.

A “real” environment is one that is based on the Python installation. This means that to get a new real environment, we must reinstall (and often rebuild) Python. This is sometimes an expensive proposition. For example, tox will rebuild an environment from scratch if any parameters are different. For this reason, virtual environments exist.

A virtual environment copies the minimum necessary out of the real environment to mislead Python into thinking that it has a new root. The exact details are not important, but what is important is that this is a simple command that just copies files around (and sometimes uses symbolic links).

There are two ways to use virtual environments: activated and unactivated. In order to use an unactivated virtual environment, which is most common in scripts and automated procedures, we explicitly call Python from the virtual environment.

This means that if we created a virtual environment in /home/name/venvs/my-special-env, we can call /home/name/venvs/my-special-env/bin/python to work inside this environment. For example, /home/name/venvs/my-special-env/bin/python -m pip will run pip but install in the virtual environment. Note that for entry-point-based scripts, they will be installed alongside Python, so we can run /home/name/venvs/my-special-env/bin/pip to install packages in the virtual environment.

The other way to use a virtual environment is to “activate” it. Activating a virtual environment in a bash-like shell means sourcing its activate script:

```
$ source /home/name/venvs/my-special-env/bin/activate
```

The sourcing sets a few environment variables, only one of which is actually important. The important variable is PATH, which gets prefixed by /home/name/venvs/my-special-env/bin. This means that commands like python or pip will be found there first. There are two cosmetic variables that get set: VIRTUAL_ENV will point to the root of the environment. This is useful in management scripts that want to be aware of virtual environments.

PS1 will get prefixed with (my-special-env), which is useful for a visual indication of the virtual environment while working interactively in the console.

In general, it is a good practice to only install third-party packages inside a virtual environment. Combined with the fact that virtual environments are “cheap,” this means that if one gets into a bad state, it is easy to just remove the whole directory and start from scratch. For example, imagine a bad package install that causes Python startup to fail. Even running pip uninstall is impossible, since pip fails on startup. However, the “cheapness” means we can remove the whole virtual environment and re-create it with a good set of packages.

Modern practice, in fact, is moving more and more toward treating virtual environments as semi-immutable: after creating them, there is a single stage of “install all required packages.” Instead of modifying it if an upgrade is required, we destroy the environment, re-create, and reinstall.

There are two ways to create virtual environments. One way is portable between Python 2 and Python 3 – virtualenv. This needs to be bootstrapped in some way, since Python does not come with virtualenv preinstalled. There are several ways to accomplish this. If Python was installed using a packaging system, such as a system packager, Anaconda, or Homebrew, then often the same system will have packaged virtualenv. If Python is installed using pyenv, in a user directory, sometimes just using pip install directly into the “original environment” is a good option, even though it is an exception to the “only install into virtual environments.” Finally, this is one of the cases pip install --user might be a good idea: this will install the package into the special “user area.” Note that this means that sometimes it will not be in $PATH, and the best way to run it will be using python-m virtualenv.

If no portability is needed, venv is a Python 3-only way of creating a virtual environment. It is accessed as python -m venv, as there is no dedicated entry point. This solves the “bootstrapping” problem of how to install virtualenv, especially when using a nonsystem Python.

Whichever command is used to create the virtual environment, it will create the directory for the environment. It is best if this directory does not exist before that. A best practice is to remove it before creating the environment. There are also options about how to create the environment: which interpreter to use and what initial packages to install. For example, sometimes it is beneficial to skip pip installation entirely. We can then bootstrap pip in the virtual environment by using get-pip.py. This is a way to avoid a bad version of pip installed in the real environment – since if it is bad enough, it cannot even be used to upgrade pip.

## 2.3 Setup and Wheels

The term “third party” (as in “third-party packages”) refers to a someone other than the Python core developers (“first party”) or the local developers (“second party”). We have covered how to install “first-party” packages in the installation section. We used pip and virtualenv to install “third-party” packages. It is time to finally turn our attention to the missing link: local development and installing local packages, or “second-party” packages.

This is an area seeing a lot of new additions, like pyproject.toml and flit. However, it is important to understand the classic way of doing things. For one, it takes a while for new best practices to settle in. For another, existing practices are based on setup.py, and so this way will continue to be the main way for a while – possibly even for the foreseeable future.

The setup.py file describes, in code, our “distribution.” Note that “distribution” is distinct from “package.” A package is a directory with (usually) __init__.py that Python can import. A distribution can contain several packages or even none! However, it is a good idea to keep a 1-1-1 relationship: one distribution, one package, named the same.

Usually, setup.py will begin by importing setuptools or distutils. While distutils is built-in, setuptools is not. However, it is almost always installed first in a virtual environment, due to its sheer popularity. Distutils is not recommended: it has not been updated for a long time. Notice that setup.py cannot meaningfully, explicitly declare it needs setuptools nor explicitly request a specific version: by the time it is read, it will have already tried to import setuptools. This non-declarativeness is part of the motivation for packaging alternatives.

The absolutely minimal setup.py that will work is the following:

```
import setuptools
setuptools.setup(
    packages=setuptools.find_packages(),
)
```

The official documentation calls a lot of other fields “required” for some reason, though a package will be built even if those are missing. For some, this will lead to ugly defaults, such as the package name being UNKNOWN.

A lot of those fields, of course, are good to have. But this skeletal setup.py is enough to create a distribution with the local Python packages in the directory.

Now, granted, almost always, there will be other fields to add. It is definitely the case that other fields will need to be added if this package is to be uploadable to a packaging index, even if it is a private index.

It is a great idea to add at least a “name” field. This will give the distribution a name. As mentioned earlier, it is almost always a good idea to name it after the single top-level package in the distribution.

A typical source hierarchy, then, will look like this:

```
setup.py
    import setuptools
    setuptools.setup(
        name='my_special_package',
        packages=setuptools.find_packages(),
    )
my_special_package/
    __init__.py
    another_module.py
    tests/
        test_another_module.py
```

Another field that is almost always a good idea is a version. Versioning software is, as always, hard. Even a running number, though, is a good way to answer the perennial question: “Is this running a newer or older version?”

There are some tools to help with managing the version number, especially assuming we want to have it also available to Python during runtime. Especially if doing Calendar Versioning, incremental is a powerful package to automate some of the tedium. bumpversion is a useful tool, especially when choosing semantic versioning. Finally, versioneer supports easy integration with the git version control system, so that a tag is all that needs to be done for release.

Another popular field in setup.py, which is not marked “required” in the documentation but is present on almost every package, is install_requires. This is how we mark other distributions that our code uses. It is a good practice to put “loose” dependencies in setup.py. This is in contrast to exact dependencies, which specify a specific version. A loose dependency looks like Twisted>=17.5 – specifying a minimum version but no maximum. Exact dependencies, like Twisted==18.1, are usually a bad idea in setup.py . They should only be used in extreme cases: for example, when using significant chunks of a package’s private API.

Finally, it is a good idea to give find_packages a whitelist of what to include, in order to avoid spurious files. For example,

```
setuptools.find_packages(include=["my_package∗"])
```

Once we have setup.py and some Python code, we want to make it into a distribution. There are several formats a distribution can take, but the one we will cover here is the wheel. If my-directory is the one that has setup.py, running pip wheel my-directory, will produce a wheel, as well as the wheels of all of its recursive dependencies.

The default is to put the wheels in the current directory, which is seldom the desired behavior. Using --wheel-dir<output-directory> will put the wheel in the directory – as well as the wheels of any distribution it depends on.

There are several things we can do with the wheel, but it is important to note that one thing we can do is pip install <wheel file>. If we add pip install <wheel file> --wheel-dir <output directory>, then pip will use the wheels in the directory and will not go out to PyPI. This is useful for reproducible installs, or support for air-gapped modes.
  
## 2.4 Tox

Tox is a tool to automatically manage virtual environments, usually for tests and builds. It is used to make sure that those run in well-defined environments and is smart about caching them in order to reduce churn. True to its roots as a test-running tool, Tox is configured in terms of test environments.

It uses a unique ini-based configuration format. This can make writing configurations difficult, since remembering the subtleties of the file format can be hard. However, in return, there is a lot of power that, while being hard to tap, can certainly help in configuring tests and build runs that are clear and concise.

One thing that Tox does lack is a notion of dependencies between build steps. This means that those are usually managed from the outside, by running specific test runs after others and sharing artifacts in a somewhat ad hoc manner.

A Tox environment corresponds, more or less, to a section in the configuration file.

```
[testenv:some-name]
.
.
.
```

Note that if the name contains pyNM (for example, py36), then Tox will default to using CPython N.M (3.6, in this case) as the Python for that test environment. If the name contains pypyNM, Tox will default to using PyPy N.M for that version – where these stand for “version of CPython compatibility,” not PyPy’s own versioning scheme.

If the name does not include pyNM or pypyNM, or if there is a need to override the default, a basepython field in the section can be used to indicate a specific Python version. By default, Tox will look for these Pythons to be available in the path. However, if the plug-in tox-pyenv is installed, Tox will query pyenv if it cannot find the right Python on the path.

As examples, we will analyze one simple Tox file and one more complicated one.

```
[tox]
envlist = py36,pypy3.5,py36-flake8
```

The tox section is a global configuration. In this example, the only global configuration we have is the list of environments.

```
[testenv:py36-flake8]
```

This section configures the py36-flake8 test environment.

```
deps =
    flake8
```

The deps subsection details which packages should be installed in the test environment’s virtual environment. Here we chose to specify flake8 with a loose dependency. Another option is to specify a strict dependency (e.g., flake8==1.0.0.). This helps with reproducible test runs. We could also specify -r <requirements file> and manage the requirements separately. This is useful if we have other tooling that takes the requirements file.
  
  ```
commands =
    flake8 useful
```

In this case, the only command is to run flake8 on the directory useful. By default, a Tox test run will succeed if all commands return a successful status code. As something designed to run from command lines, flake8 respects this convention and will only exit with a successful status code if there were no problems detected with the code.
[testenv]

The other two environments, lacking specific configuration, will fall back to the generic environment. Note that because of their names, they will use two different interpreters: CPython 3.6 and PyPy with Python 3.5 compatibility.

```
deps =
    pytest
```

In this environment , we install the pytest runner. Note that in this way, our tox.ini documents the assumptions on the tools needed to run the tests. For example, if our tests used Hypothesis or PyHamcrest, this is where we would document it.

```
commands =
    pytest useful
```

Again, the command run is simple. Note, again, that pytest respects the convention and will only exit successfully if there were no test failures.

As a more realistic example, we turn to the tox.ini of ncolony:

```
[tox]
envlist = {py36,py27,pypy}-{unit,func},py27-lint,py27-wheel,docs
toxworkdir = {toxinidir}/build/.tox
```

We have more environments. Note that we can use the {} syntax to create a matrix of environments. This means that {py36,py27,pypy}-{unit,func} creates `3*2=6` environments. Note that if we had a dependency that made a “big jump” (for example, Django 1 and 2), and we wanted to test against both, we could have made {py36,py27, pypy}-{unit,func}-{django1,django2}, for a total of `3*2*2=12` environments. Notice the numbers for a matrix test like this climb up fast – and when using an automated test environment, it means things would either take longer or need higher parallelism.

This is a normal trade-off between comprehensiveness of testing and resource use. There is no magical solution other than to carefully consider how many variations to officially support.

```
[testenv]
```

Instead of having a testenv per variant, we choose to use one test environment but special case the variants by matching. This is a fairly efficient way to create many variants of test environments.

```
deps =
    {py36,py27,pypy}-unit: coverage
    {py27,pypy}-lint: pylint==1.8.1
    {py27,pypy}-lint: flake8
    {py27,pypy}-lint: incremental
    {py36,py27,pypy}-{func,unit}: Twisted
```

We need coverage only for the unit tests, while Twisted is needed for both the unit and functional tests. The pylint strict dependency ensures that as pylint adds more rules, our code does not acquire new test failures. This does mean we need to update pylint manually from time to time.

```
commands =
    {py36,py27,pypy}-unit: python -Wall \
                                  -Wignore::DeprecationWarning \
                                  -m coverage \
                                  run -m twisted.trial \
                                  --temp-directory build/_trial_temp \
                                  {posargs:ncolony}
    {py36,py27,pypy}-unit: coverage report --include ncolony∗ \
                           --omit ∗/tests/∗,∗/interfaces∗,∗/_version∗ \
↪                     --show-missing --fail-under=100
    py27-lint: pylint --rcfile admin/pylintrc ncolony
    py27-lint: python -m ncolony tests.nitpicker
    py27-lint: flake8 ncolony
    {py36,py27,pypy}-func: python -Werror -W ignore::DeprecationWarning \
                                  -W ignore::ImportWarning \
                                  -m ncolony tests.functional_test
```

Configuring “one big test environment” means we need to have all our commands mixed in one bag and select based on patterns. This is also a more realistic test run command – we want to run with warnings enabled, but disable warnings we do not worry about, and also enable code coverage testing. While the exact complications will vary, we almost always need enough things so that the commands will grow to a decent size.

```
[testenv:py27-wheel]
skip_install = True
deps =
      coverage
      Twisted
      wheel
      gather
commands =
      mkdir -p {envtmpdir}/dist
      pip wheel . --no-deps --wheel-dir {envtmpdir}/dist
      sh -c "pip install --no-index {envtmpdir}/dist/∗.whl"
      coverage run {envbindir}/trial \
               --temp-directory build/_trial_temp {posargs:ncolony}
      coverage report --include ∗/site-packages/ncolony∗ \
                      --omit ∗/tests/∗,∗/interfaces∗,∗/_version∗ \
                      --show-missing --fail-under=100
```

The py27-wheel test run ensures we can build, and test, a wheel. As a side effect, this means a complete test run will build a wheel. This allows us to upload a tested wheel to PyPI when it is release time.

```
[testenv:docs]
changedir = docs
deps =
    sphinx
    Twisted
commands =
    sphinx-build -W -b html -d {envtmpdir}/doctrees . {envtmpdir}/html
basepython = python2.7
```
The documentation build is one of the reasons why Tox shines. It only installs sphinx in the virtual environment for building documentation. This means that an undeclared dependency on sphinx would make the unit tests fail, since sphinx is not installed there.

## 2.5 Pipenv and Poetry

Pipenv and Poetry are two new ways to produce Python projects. They are inspired by tools like yarn and bundler for JavaScript and Ruby, respectively, which aim to encode a more complete development flow. By themselves, they are not a replacement for Tox – they do not encode the ability to run with multiple Python interpreters, or completely override the dependency. However, it is possible to use them, in tandem with a CI-system configuration file, like Jenkinsfile or .circleci/config.yml, to build against multiple environments.

However, their main strength is in allowing easier interactive development. This is useful, sometimes, for more exploratory programming.

### 2.5.1 Poetry

The easiest way to install poetry is to use pip install --user poetry. However, this will install all of its dependencies into your user environment, which has the potential to make a mess of things. One way to do it in a clean way is to create a dedicated virtual environment.

```
$ python3 -m venv ~/.venvs/poetry
$ ~/.venvs/poetry/bin/pip install poetry
$ alias poetry=~/.venvs/poetry/bin/poetry
```

This is an example of using an unactivated virtual environment.

The best way to use poetry is to create a dedicated virtual environment for the project. We will build a small demo project. We will call it “useful.”

```
$ mkdir useful
$ cd useful
$ python3 -m venv build/useful
$ source build/useful/bin/activate
(useful)$ poetry init
(useful)$ poetry add termcolor
(useful)$ mkdir useful
(useful)$ touch useful/__init__.py
(useful)$ cat > useful/__main__.py
import termcolor
print(termcolor.colored("Hello", "red"))
```

If we have done all this, running python -m useful in the virtual environment will print a red Hello. After we have interactively tried various colors, and maybe decided to make the text bold, we are ready to release:

```
(useful)$ poetry build
(useful)$ ls dist/
useful-0.1.0-py2.py3-none-any.whl useful-0.1.0.tar.gz
```

### 2.5.2 Pipenv

Pipenv is a tool to create virtual environments that match a specification, in addition to ways to evolve the specification. It relies on two files: Pipfile and Pipfile.lock. We can install pipenv similarly to how we installed poetry – in a custom virtual environment and add an alias.

In order to start using it, we want to make sure no virtual environments are activated. Then,

```
$ mkdir useful
$ cd useful
$ pipenv add termcolor
$ mkdir useful
$ touch useful/__init__.py
$ cat > useful/__main__.py
import termcolor
print(termcolor.colored("Hello", "red"))
$ pipenv shell
(useful-hwA3o_b5)$ python -m useful
```

This will leave in its wake a Pipfile that looks like this:

```
[[source]]
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"
[packages]
termcolor = "∗"
[dev-packages]
[requires]
python_version = "3.6"
```

Note that in order to package useful , we still have to write a setup.py. Pipenv limits itself to managing virtual environments, and it does consider building and publishing separate tasks.

## 2.6 DevPI

DevPI is a PyPI-compatible server that can be run locally. Though it does not scale to PyPI-like levels, it can be a powerful tool in a number of situations.

DevPI is made up of three parts. The most important one is devpi-server. For many use cases, this is the only part that needs to run. The server serves, first and foremost, as a caching proxy to PyPI. It takes advantage of the fact that packages on PyPI are immutable: once we have a package, it can never change.

There is also a web server that allows us to search in the local package directory. Since a lot of use cases do not even involve searching on the PyPI website, this is definitely optional. Finally, there is a client command-line tool that allows configuring various parameters on the running instance. The client is most useful in more esoteric use cases.

Installing and running DevPI is straightforward. In a virtual environment, simply run:

```
(devpi)$ pip install devpi-server
(devpi)$ devpi-server --start --init
```

The pip tool, by default, goes to pypi.org. For some basic testing of DevPI, we can create a new virtual environment, playground, and run:

```
(playground)$ pip install \
              -i http://localhost:3141/root/pypi/+simple/ \
              httpie glom
(playground)$ http --body https://httpbin.org/get | glom '{"url":"url"}'
{
  "url": "https://httpbin.org/get"
}
```

Having to specify the -i ... argument to pip every time would be annoying. After checking that everything worked correctly, we can put the configuration in an environment variable:

```
$ export PIP_INDEX_URL=http://localhost:3141/root/pypi/+simple/
```

Or to make things more permanent:

```
$ mkdir -p ~/.pip && cat > ~/.pip/pip.conf << EOF
[global]
index-url = http://localhost:3141/root/pypi/+simple/
[search]
index = http://localhost:3141/root/pypi/
```

The above file location works for UNIX operating systems. On Mac OS X the configuration file is $HOME/Library/Application Support/pip/pip.conf. On Windows the configuration file is %APPDATA%\pip\pip.ini.

DevPI is useful for disconnected operations. If we need to install packages without a network, DevPI can be used to cache them. As mentioned earlier, virtual environments are disposable and often treated as mostly immutable. This means that a virtual environment with the right packages is not a useful thing without a network. The chances are high that some situation or the other will either require or suggest creating it from scratch.

However, a caching server is a different matter. If all package retrieval is done through a caching proxy, then destroying a virtual environment and rebuilding it is fine, since the source of truth is the package cache. This is as useful for taking a laptop into the woods for disconnected development as it is for maintaining proper firewall boundaries and having a consistent record of all installed software.

In order to “warm up” the DevPI cache , that is, make sure it contains all needed packages, we need to use pip to install them. One way to do it is, after configuring DevPI and pip, is to run tox against a source repository of software under development. Since tox goes through all test environments, it downloads all needed packages.

It is definitely a good practice to also preinstall in a disposable virtual environment any requirements.txt that are relevant.

However, the utility of DevPI is not limited to disconnected operations. Configuring one inside your build cluster, and pointing the build cluster at it, completely avoids the risk for a “leftpad incident,” where a package you rely on gets removed by the author from PyPI. It might also make builds faster, and it will definitely cut out a lot of outgoing traffic.

Another use for DevPI is to test uploads, before uploading them to PyPI. Assuming devpi-server is already running on the default port, we can:

```
(devpi)$ pip install devpi-client twine
(devpi)$ devpi use http://localhost:3141
(devpi)$ devpi user -c testuser password=123
(devpi)$ devpi login testuser --password=123
(devpi)$ devpi index -c dev bases=root/pypi
(devpi)$ devpi use testuser/dev
(devpi)$ twine upload --repository http://localhost:3141/testuser/dev \
               -u testuser -p 123 my-package-18.6.0.tar.gz
(devpi)$ pip install -i http://localhost:3141/testuser/dev my-package
```

Note that this allows us to upload to an index that we only use explicitly, so we are not shadowing my-package for all environments that are not using this explicitly.

An even more advanced use-case, we can do this:

```
(devpi)$ devpi index root/pypi mirror_url=https://ourdevpi.local
```

This will make our DevPI server a mirror of a local, “upstream,” DevPI server. This allows us to upload private packages to the “central” DevPI server, in order to share with our team. In those cases, the upstream DevPI server will often need to be run behind a proxy – and we need to have some tools to properly manage user access.

Running a “centralized” DevPI behind a simple proxy that asks for username and password allows an effective private repository. For that, we would first want to remove the root/pypi index:

```
$ devpi index --delete root/pypi
```

and then re-create it with

```
$ devpi index --create root/pypi
```

This means the root index no longer will mirror pypi . We can upload packages now directly to it. This type of server is often used with the argument --extra-index-url to pip, to allow pip to retrieve both from the private repository and the external one. However, sometimes it is useful to have a DevPI instance that only serves specific packages. This allows enforcing rules about auditing before using any packages. Whenever a new package is needed, it is downloaded, audited, and then added to the private repository.

## 2.7 Pex and Shiv

While it is currently nontrivial to compile a Python program into one self-contained executable, we can do something that is almost as good. We can compile a Python program into a single file that only needs an installed interpreter to run. This takes advantage of the particular way Python handles startup.
When running python /path/to/filename, Python does two things:

- Adds the directory /path/to to the module path.

- Executes the code in /path/to/filename.

When running python/path/to/directory/, Python will behave exactly as though we typed python/path/to/directory/__main__.py.

In other words, Python will do the following two things:

- Add the directory /path/to/directory/ to the module path.

- Executes the code in /path/to/directory/__main__.py.

When running python /path/to/filename.zip, Python will treat the file as a directory.
In other words, Python will do the following two things:

- Add the “directory” /path/to/filename.zip to the module path.

- Executes the code in the __main__.py it extracts from /path/to/filename.zip.

Zip is an end-oriented format: The metadata, and pointers to the data, are all at the end. This means that adding a prefix to a zip file does not change its contents.

So, if we take a zip file, and prefix it with #!/usr/bin/python<newline>, and mark it executable, then when running it, Python will be running a zip file. If we put the right bootstrapping code in __main__.py, and put the right modules in the zip file, we can get all of our third-party dependencies in one big file.

Pex and Shiv are tools for producing such files, but they both rely on the same underlying behavior of Python and of zip files.

### 2.7.1 Pex

Pex can be used either as a command-line tool or as a library. When using it as a command-line tool, it is a good idea to prevent it from trying to do dependency resolution against PyPI. All dependency resolution algorithms are flawed in some way. However, due to pip’s popularity, packages will explicitly work around flaws in its algorithm. Pex is less popular, and there is no guarantee that packages will try explicitly to work with it.

The safest thing to do is to use pip wheel to build all wheels in a directory and then tell Pex to use only this directory.

For example,

```
$ pip wheel --wheel-dir my-wheels -r requirements.txt
$ pex -o my-file.pex --find-links my-wheels --no-index \
      -m some_package
```

Pex has a few ways to find the entry point. The two most popular ones are -m some_package, which will behave as though python -m some_package; or -c console-script, which will find what script would have been installed as console-script, and invoke the relevant entry point.

It is also possible to use Pex as a library.

```
from pex import pex_builder
```

Most of the logic to build Pex files is in the pex_builder module.

```
builder = pex_builder.PEXBuilder()
```

We create a builder object.

```
builder.set_entry_point('some_package')
```

We set the entry point. This is equivalent to the -m some_package argument on the command line.

```
builder.set_shebang(sys.executable)
```

The Pex binary has a sophisticated argument to determine the right shebang line. This is sometimes specific to the expected deployment environment, so it is a good idea to put some thought into the right shebang line. One option is /usr/bin/env python, which will find what the current shell calls python. It is sometimes a good idea to specify a version here, such as /usr/local/bin/python3.6, for example.

```
subprocess.check_call([sys.executable, '-m', 'pip', 'wheel',
                       '--wheel-dir', 'my-wheels',
                       '--requirements', 'requirements.txt'])
```

Once again, we create wheels with pip. As tempting as it is, pip is not usable as a library, so shelling out is the only supported interface.

```
for dist in os.listdir('my-wheels'):
    dist = os.path.join('my-wheels', dist)
    builder.add_dist_location(dist)
```

We add all packages that pip built.

```
builder.build('my-file.pex')
```

Finally, we have the builder produce a Pex file.

### 2.7.2 Shiv

Shiv is a modern take on the same ideas behind Pex. However, since it uses pip directly, it needs to do a lot less itself.

```
$ shiv -o my-file.shiv -e some_package -r requirements.txt
```

Because shiv just offloads to pip actual dependency resolution, it is safe to call it directly. Shiv is a younger alternative to Pex. This means a lot of cruft has been removed, but it is still lacking somewhat in maturity.

For example, the documentation for command-line arguments is a bit thin. There is also no way to currently use it as a library.
2.8 XAR

XAR (eXecutable ARchive) is a generic format for shipping self-contained executables. While not being Python specific, it is designed as Python first. It is natively installable via PyPI, for example.

The downsides of XAR is that it assumes a certain level of system support for fuse (Filesystem in User SpacE) that is not universal yet. This is not a problem if all machines designed to run the XAR, Linux, or Mac OS X are under your control. The instructions for how to install proper FUSE support are not complex, but they do require administrative privileges. Note that XAR is also less mature than Pex.

However, assuming proper SquashFS support, many other concerns vanish: including, most importantly, compared to pex or shiv, local Python versions. This makes XAR an interesting choice for either shipping developer tools or local system management scripts.

In order to build a XAR, we can call setup.py with bdist_xar, if xar is installed.

```
python setup.py bdist_xar --console-scripts=my-script
```

In this example, my-script is the name of a console script entry point, specified in the setup.py with the following:

```
entry_points=dict(
   console_scripts=["my-script = package.module:function"],
)
```
In some cases, the --console-scripts argument is not necessary. If, as in the example above, there is only one console script entry point, then it is implied. Otherwise, if there is a console script with the same name as the package, then that one is used. This accounts for quite a few cases, which means this argument is often redundant.
