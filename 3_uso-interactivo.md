<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_3
-->

# Uso interactivo

Python is often used for exploratory programming. Often, the final result is not the program but an answer to a question. For scientists, the question might be “what is the likelihood of a medical intervention working?” For people troubleshooting computers, the question might be “which log file has the message I need?”

However, regardless of the question, Python can often be a powerful tool to answer it. More importantly, in exploratory programming, we expect to encounter more questions, based on the answer.

The interactive model in Python comes from the original Lisp environments’ “Read-Eval-Print Loop” (REPL, for short). The environment reads a Python expression, evaluates it in an environment that persists in memory, prints the result, and loops back.

The REPL environment native to Python is popular because it is built in. However, a few third-party REPL tools are even more powerful, built to do things that the native one could not or would not. These tools give a powerful way to interact with the operating system, exploring and molding until the desired state is achieved.

## 3.1 Native Console

Launching python without any arguments will open up the “interactive console .” It is a good idea to use pyenv or a virtual environment to make sure that the Python version is up to date.

The availability of an interactive console immediately, without installing anything else, is one reason why Python is suited for exploratory programming. We can immediately ask questions.

The questions can be trivial:

```
>>> 2 + 2
4
```

They can be used to calculate sales tax in the Bay area:

```
>>> rate = 9.25
>>> price = 5.99
>>> after_tax = price ∗ (1 + rate / 100.)
>>> after_tax
6.544075
```

Or they can answer important questions about the operating environment:

```
>>> import os
>>> os.path.isfile(os.path.expanduser("~/.bashrc"))
True
```

Using the Python native console without readline is unpleasant. Rebuilding Python with readline support is a good idea, so that the native console will be useful. If this is not an option, using one of the alternative consoles is recommended. For example, a locally built Python with specific options might not include readline, and it might be problematic to redistribute a new Python to the entire team.

If readline support is installed, Python will use it to support line editing and history. It is also possible to save the history using readline.write_history_file. This is often after having used the console for a while, in order to have a reference for what has been done, or to copy whatever ideas worked into a more permanent form.

When using the console, the _ variable will have the value of the last expression-statement evaluated. Note that exceptions, statements that are not expressions and statements stat are expressions that evaluate to None will not change the value of `_`. This is useful during an interactive session when only after having seen the representation of the value, do we realize we needed that as an object.

```
>>> import requests
>>> requests.get("http://en.wikipedia.org")
<Response [200]>
>>> a=_
>>> a.text[:50]
'<!DOCTYPE html>\n<html class="client-nojs" lang="en'
```

Only after using the .get function, we realize that what we actually wanted was the text. Luckily, the Response object is saved in the variable _. We put the value of the variable in a immediately, _ is replaced quickly. As soon as we evaluate a.text[:50], _ is now a 50-character string. If we had not saved _ in a variable, all but the first 50 characters would have been lost.

Notice that this _ convention is kept by every good Python REPL, and so the trick to “keep returned values in one-letter variables” is often useful when doing explorations.

## 3.2 The Code Module

The code module allows us to run our own interactive loop. An example of when this can be useful is when running commands with a special flag, we can drop into a prompt at a specific point. This allows us to have a REPL environment after we have set things up in a certain way. This holds for both inside the interpreter, setting up the namespace with useful things; and in the external environment, perhaps initializing files or setting up external services.

The highest-level use of code is the interact function.

```
>>> import code
>>> code.interact(banner="Welcome to the special interpreter",
...               local=dict(special=[1, 2, 3]))
Welcome to the special interpreter
>>> special
[1, 2, 3]
>>> ^D
now exiting InteractiveConsole...
>>> special
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'special' is not defined
```
  
This shows an example of running a REPL loop with a variable special, which sets to a short list.

For the lowest-level use of code, if you want to own the UI yourself, code.compile_command(source, [filename="<input>"], symbol="single") will return a code object (that can be passed to exec), None if the command is incomplete or raise SyntaxError, OverflowError, or ValueError if there is a problem with the command.

The symbol argument should almost always be "single." The exception is if the user is prompted to enter code that will evaluate to an expression (for example, if the value is to be used by the underlying system). In that case, symbol should be set to "eval."

This allows us to manage interacting with the user ourselves. It can be integrated with a UI, or a remote network interface, to allow interactivity in any environment.

## 3.3 ptpython

The ptpython tool , short for “prompt toolkit Python,” is an alternative to the built-in REPL. It uses the prompt toolkit for console interaction, instead of readline.

Its main advantage is the simplicity of installation. A simple pip install ptpython in a virtual environment and regardless of readline build problems, a high-quality Python REPL appears.

ptpython supports completion suggestions, multiline editing, and syntax highlighting.

On startup, it will read ~/.ptpython/config.py. This means it is possible to locally customize ptpython in arbitrary ways. The way to configure is to implement a function, configure, which accepts an object (of type PythonRepl) and mutates it.

There are a lot of possibilities, and sadly the only real documentation is the source code. The relevant reference __init__ is ptpython.python_input.PythonInput. Note that config.py really is an arbitrary Python file. Therefore, if you want to distribute modifications internally, it is possible to distribute a local PyPI package and have people import a configure function from it.

## 3.4 IPython

IPython, which is also the foundation of Jupyter, which will be covered later, is an interactive environment whose roots are in the scientific computing community. IPython is an interactive command prompt, similar to the ptpython utility or Python’s native REPL.

However, it aims to give a sophisticated environment. One of the things that it does is number every input and output from the interpreter. It is useful to be able to refer to those numbers later. IPython puts all inputs in the In array and outputs in the Out array. This allows for nice symmetry: if IPython says In[4], for example, this is how to access that value.

```
In [1]: print("hello")
hello
In [2]: In[1]
Out[2]: 'print("hello")'
In [3]: 5 + 4.0
Out[3]: 9.0
In [4]: Out[3] == 9.0
Out[4]: True
```

It also supports tab completion out of the box. IPython uses both its own completion and the jedi library for static completion.

It also supports built-in help. Typing var_name? will try to find the best context-relevant help for the object in the variable, and display it. This works for functions, classes, built-in objects, and more.

```
In [1]: list?
Init signature: list(self, /, ∗args, ∗∗kwargs)
Docstring:
list() -> new empty list
list(iterable) -> new list initialized from iterable's items
Type:           type
```

IPython also supports something called “magic,” where prefixing a line with % will execute a magic function. For example, %run will run a Python script inside the current namespace. As another example, %edit will launch an editor. This is useful if during usage, a statement needs more sophisticated editing.

In addition, prefixing a line with ! will run a system command. One useful way to take advantage of this is !pip install something. This is why it is useful to install IPython inside virtual environments that are used for interactive development.

IPython can be customized in a number of ways. While in an interactive session, the %config magic command can be used to change any option. For example, %config InteractiveShell.autocall = True will set the autocall option, which means expressions that are callable are called, even without parentheses. This is moot for any options that only affect startup. We can change these options, as well as any others, using the command line. For example, ipython --InteractiveShell.autocall=True, will launch into an autocalling interpreter.

If we want custom logic to decide on configuration, we can run IPython from a specialized Python script.

```
from traitlets import config
import IPython
my_config = config.Config()
my_config.InteractiveShell.autocall = True
IPython.start_ipython(config=my_config)
```

If we package this in a dedicated Python package, we can distribute it to a team using either PyPI or a private package repository. This allows having a homogenous custom IPython configuration for a development team.

Finally, configuration can also be encoded in profiles, which are Python snippets located under ~/.ipython by default. The profile directory can be modified by an explicit command-line parameter --ipython-dir or an environment variable IPYTHONDIR.

## 3.5 Jupyter Lab

Jupyter is a project that uses web-based interaction to allow for sophisticated exploratory programming. It is not limited to Python, though it does originate in Python. The name stands for “Julia/Python/R,” the three languages most popular for exploratory programming, especially in data science.

Jupyter Lab, the latest evolution of Jupyter, was originally based on IPython. It now sports a full-featured web interface and a way to remotely edit files. The main users of Jupyter tend to be scientists. They take advantage of the ability to see how results were derived to add in reproducibility and peer review.

Reproducibility and peer review are also important for DevOps work. The ability to show the steps that led to deciding which list of hosts to restart, so that it can be regenerated if circumstances change, for example, is highly useful. The ability to attach a notebook, detailing the steps that were taken during an outage, together with the output from the steps, to a postmortem analysis can aid in understanding what happened, and how to avoid a problem in the future or recover from it more effectively.

It is important to note here that notebooks are not an auditability tool: they can be executed out of order and have blocks modified and re-executed. However, properly used, they allow us to record what has been done.

Jupyter allows true exploratory programming. This is useful for scientists, who might not understand the true scope of a problem beforehand.

It is important to note here that notebooks are not an auditability tool: they can be executed out of order and have blocks modified and re-executed. However, properly used, they allow us to record what has been done. This is also useful for systems integrators, faced with complex systems, where it is hard to predict where the problem lies before exploration, either.

Installing Jupyter Lab in a virtual environment is a simple matter of doing pip install jupyterlab. When starting jupyter lab, by default, it will start a web server on an open port starting at 8888 and attempt to launch a web browser to watch it. If working on an environment that is “too interesting” (for example, the default web browser is not configured properly), the standard output will contain a preauthorized URL to access the server. If all else fails, it is possible to copy-paste the token printed to standard output into the browser after manually entering the URL in a web browser. It is also possible to access the token with jupyter notebook list, which will list all currently running servers.

Once inside Jupyter Lab, there are five things we can launch:

- Console

- Terminal

- Text editor

- Notebook

- Spreadsheet editor

The Console is a web-based interface to IPython. All that was said about IPython previously (for example, the In and Out arrays). The Terminal is a full-fledged terminal emulator in the browser. This is useful for a remote terminal inside a VPN: all it needs as far as connectivity needs is an open web port, and it can also be protected in the regular ways that web ports are protected: TLS, client-side certificates, and more. The text editor is useful for editing remote files. This is an alternative to running a remote shell, and an editor such as vi in it. It has the advantage of avoiding UI lag, while still having full file-editing capabilities.

The most interesting thing to launch, though, is a notebook: indeed, many a session will using nothing but notebooks. A notebook is a JSON file that records a session. As the session unfolds, Jupyter will save “snapshots” of the notebook, as well as the latest version. A notebook is made of a sequence of Cells. The two most popular cell types are “code” and “Markdown.” A “code” cell type will contain a Python code snippet. It will execute it in the context of the session’s namespace. The namespace is persistent from one cell execution to the other, corresponding to a “kernel” running. The kernel accepts cell content using a custom protocol, interprets them as Python, executes them, and returns both whatever was returned by the snippet as well as output from this.

When launching a Jupyter server, by default, it will use the local IPython kernel as its only possible kernel. This means that the server will, for example, only be able to use the same Python version and the same set of packages. However, it is possible to connect a kernel from a different environment to this server. The only requirement is that the environment has the ipykernel package installed. From the environment, run:

```
python -m ipykernel install \
       --name my-special-env \
       --display-name "My Env"
       --prefix=$DIRECTORY
```

Then, from the Jupyter server environment, run:

```
jupyter kernelspec install $DIRECTORY/jupyter/kernels/my-special-env
```

This will cause the Jupyter server in this environment to support the kernel from the special environment. This allows running one semipermanent Jupyter server and connecting kernels from any environment that is “interesting”: installing specific modules, running a specific version of Python, or any other difference. One other usage of alternative kernels, which will not be covered here in details, is alternative languages. Julia and R kernels are supported upstream, but there exist third-party kernels for many languages – even bash!

Jupyter supports all magic commands from IPython. Especially useful, again, is the !pip install ... command to install new packages in the virtual environment. Especially if being careful, and installing precise dependencies, this makes a notebook be high-quality documentation of how to achieve a result in a way that is replayable.

Since Jupyter is one level of indirection away from the kernel, we can restart the kernel directly from Jupyter. This means the whole Python process gets restarted, and any in-memory results are gone. We can re-execute cells in any order, but there is a single-button way to execute all cells in order. Restarting the kernel, and executing all cells in order, is a nice way of “testing” a notebook for working conditions – although, naturally, any effects on the external world will not be reset.

Jupyter notebooks are useful as attachments to tickets and postmortems, both as a way of documenting specific remediations, as well as documenting “state of things” by running query APIs and collecting the results in the notebook. Usually, when attaching a notebook in such a way, it is useful to also export it to a more easily readable format, such as HTML or PDF, and attach that as well. However, more and more tools integrate direct notebook viewing, making this step redundant. For example, GitHub projects and Gists already render notebooks directly.

Alongside the notebooks, Jupyter Lab sports a rudimentary, but functional, browser-based remote development environment. The first part is a remote file manager. Among other things, this allows uploading and downloading files. One use for it, among many, is the ability to upload notebooks from the local computer and download them back again. There are better ways to manage notebooks, but in a pinch, being able to retrieve a notebook is extremely useful. Similarly, any persistent outputs from Jupyter, such as processed data files, images, or charts, can also be downloaded.

Next, alongside the notebooks, is a remote IPython console. Though of limited use next to the notebook, there are still some cases where using the console is easier. A session that requires a lot of short commands can be more keyboard-centric by using the IPython console, and thus more efficient.

There is also a file editor. Although it is a far cry from being a full-fledged developer editor, lacking thorough code understanding and completion, it is often useful in a pinch. It allows directly editing files on the remote Jupyter host. One use case is directly fixing library code that the notebook is using, and then restarting the kernel. While integrating it into a development flow takes some care, as an emergency measure to fix and continue, this is invaluable.

Last, there is a remote browser-based terminal. Between the terminal, the file editor and the file manager, a running Jupyter server allows complete browser-based remote access and management, even before thinking of the notebooks. This is important to remember for security implications, but it is also a powerful tool whose various uses we will explore later. For now, suffice it to say, the power that using a Jupyter notebook brings to remote system administration tasks is hard to overestimate.
