<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_4
-->

# Automatización de SO

Python was initially built to automate a distributed operating system called an “amoeba.” Though the Amoeba OS is mostly forgotten, Python has found a home in automating UNIX-like operating systems tasks.

Python wraps the traditional UNIX C API lightly, giving full access to the system calls that run UNIX while making them just a little safer to use: an approach that was dubbed “C with foam padding.” This willingness to wrap low-level operating system APIs has made it a good choice for the wide berth between the programs that UNIX shell is good for, and the programs the C programming language is good for.

As the saying goes, with great power comes great responsibility. In order to allow for programmer power and flexibility, Python does not stop programmers from wreaking havoc. Carefully using Python to write programs that work and, more importantly, break in predictable, safe ways is a skill that is worth mastering.

## 4.1 Files

 It has been a long time since “everything is a file” was an accurate mantra on UNIX. Nevertheless, many things are files, and even more things are enough like files that manipulating them with file-based system calls works.

When dealing with files’ contents, Python programs can go down one of two routes. They can open them as “text” or as “binary.” Although files themselves are neither text nor binary, just a blob of bytes, the opening mode is important.

When opening a file as binary, the bytes are read and written as is – as byte strings. This is useful with files that are non-textual, such as picture files.

When opening a file as text, an encoding has to be used. It can be specified explicitly, but in certain situations, defaults apply. All bytes read from the file are decoded, and the code receives a character string. All strings written to the file are encoded to bytes. This means the interface with the file is with strings – sequences of characters.

A simple example of a binary file is the GIMP “XCF” internal format. GIMP is an image manipulation program, and it saves files in its internal XCF format with more details than images have. For example, layers in the XCF will be separate, for easy editing.

```
>>> with open("Untitled.xcf", "rb") as fp:
...     header = fp.read(100)
```

Here we open a file. The rb argument stands for “read, binary.” We read the first hundred bytes. We will need far fewer, but this is often a useful tactic. Many files have some metadata at the beginning.

```
>>> header[:9].decode('ascii')
'gimp xcf '
```

The first nine characters can actually be decoded to ASCII text, and happen to be the name of the format.

```
>>> header[9:9+4].decode('ascii')
'v011'
```

The next four characters are the version. This file is the 11th version of XCF.

```
>>> header[9+4]
0
```

A 0 byte finishes the “what is this file” metadata. This has various advantages.

```
>>> struct.unpack('>I', header[9+4+1:9+4+1+4])
(1920,)
```

The next four bytes are the width, as a number in big-endian format. The struct module knows how to parse these. The > says it is big endian, and the I says it is an unsigned 4-byte integer.

```
>>> struct.unpack('>I', header[9+4+1+4:9+4+1+4+4])
(1080,)
```
The next four bytes are the width. This simple code gave us the high-level data: it confirmed that this is XCF, it showed what version of the format it is, and we could see the dimensions of the image.

When opening files as text, the default encoding is UTF-8. One advantage of UTF-8 is that it is designed to fail quickly if something is not UTF-8: it is carefully designed to fail on ISO-8859-[1-9], which predates Unicode, as well as on most binary files. It is also backwards compatible with ASCII, which means pure ASCII files will still be valid UTF-8.

The most popular way to parse text files is line by line, and Python supports that by having an open text file be an iterator that yields the lines in order.

```
>>> fp = open("things.txt", "w")
>>> fp.write("""\
... one line
... two lines
... red line
... blue line
... """)
39
>>> fp.close()
>>> fpin = open("things.txt")
>>> next(fpin)
'one line\n'
>>> next(fpin)
'two lines\n'
>>> next(fpin)
'red line\n'
>>> next(fpin)
'blue line\n'
>>> next(fpin)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```
 
Usually we will not call next directly, but use for. Additionally, usually we use files as context managers, to make sure they close at a well-understood point. However, especially in REPL scenarios, there is a trade-off: opening the file without a context manager allows us to explore reading bits and pieces.

Files on a unix system are more than just blobs of data. They have various metadata attached, which can be queried and sometimes changed.

The rename system call is wrapped in the os.rename Python function. Since rename is atomic, this can help implement operations that require a certain state.

In general, note that the os module tends to be a thin wrapper over operating system calls. The discussion here is relevant to UNIX-like systems: Linux, BSD-based systems, and, for the most part, Mac OS X. It is worth keeping in mind, but it is not worth pointing each place where we are making UNIX-specific assumptions.

For example,

```
with open("important.tmp", "w") as fout:
    fout.write("The horse raced past the barn")
    fout.write("fell.\n")
os.rename("important.tmp", "important")
```

This ensures that when reading the important file , we do not accidentally misunderstand the sentence. If the code crashes in the middle, instead of believing that the horse raced past the barn, we get nothing from important. We only rename important.tmp to important at the end, after the last word has been written to the file.

The most important example of a file-which-is-not-a-blob, in UNIX, is a directory. The os.makedirs function allows us to ensure a directory exists easily with
os.makedirs(some_path, exists_ok=True)

This combines powerfully with the path operations from os.path to allow safe creation of a nested file:

```
def open_for_write(fname, mode=""):
    os.makedirs(os.path.dirname(fname), exists_ok=True)
    return open(fname, "w" + mode)
    
with open_for_write("some/deep/nested/name/of/file.txt") as fp:
    fp.write("hello world")
```

This can come in useful, for example, when mirroring an existing file layout.

The os.path module has, mostly, string manipulation functions that assume strings are file names. The dirname function returns the directory name, so os.path.dirname("a/b/c") would return a/b. Similarly, the function basename returns the “file name,” so os.path.basename("a/b/c") would return c. The inverse of both is the os.path.join function, which join paths: os.path.join("some", "long/and/winding", "path") would return some/long/and/finding/path.

Another set of functions in the os.path module has a slightly higher-level abstraction for getting file metadata. It is important to note that these functions are often light wrappers around operating system functionality, and do not try to hide operating system quirks. This means that operating system quirks can “leak” through the abstraction.

The biggest metadata is os.path.exists: does the file exist? This comes in handy sometimes, though often it is better to write code in a way that is agnostic of file existence: file existence can have races. Subtler are the os.path.is... functions: isdir, isfile, islink, and more can decide if a file name points to what we expect.

The os.path.get... functions get non-boolean metadata: access time, modification time, c-time (sometimes shortened to “creation time,” but misleadingly not the actual time of creation in a set of subtle circumstances, and more accurately refer to as “i-node modification time”), and getsize getting the size of the file.

The shutil module (“shell utilities”) contains some higher-level operations. shutil.copy will copy a file’s contents as well as metadata. shutil.copyfile will copy contents only. shutil.rmtree is the equivalent of rm -r, while shutil.copytree is the equivalent of cp -r.

Finally, temporary files are often useful. Python’s tempfile module produces temporary files that are secure and resistant to leaks. The most useful functionality is NamedTemporaryFile , which can be used as a context.

A typical usage looks like this:

```
with NamedTemporaryFile() as fp:
    fp.write("line 1\n")
    fp.write("line 2\n")
    fp.flush()
    function_taking_file_name(fp.name)
```

Note that the fp.flush there is important. The file object caches write until closed. However, NamedTemporaryFile will vanish when closed. Explicitly flushing it is important before calling a function that will reopen the file for reading.

## 4.2 Processes

The main module to deal with running subprocesses in Python is subprocess. It contains a high-level abstraction that matches the intuitive model most have when they think of “running commands,” rather than the low-level model implemented in UNIX, using exec and fork.

It is also a powerful alternative to calling the os.system function , which is problematic in several ways. For one, os.system spawns an extra process, the shell. This means that it depends on the shell, which on some weirder installation can differ with a more “exotic” system shell like ash or fish. Finally, it means that the shell will parse the string, which means the string has to be properly serialized. This is a hard task to do, since the formal specification for the shell parser is long. Unfortunately, it is not hard to write something that will work fine most of the time, so most bugs are subtle and break at the worst possible time. This sometimes even manifests as a security flaw.

While subprocess is not completely flexible, for most needs, this module is perfectly adequate.

subprocess itself is also divided into high-level functions and a lower-level implementation level. The high-level functions, which should be used in most circumstances, are check_call and check_output. Among other benefits, they behave like running a shell with -e, or set err – they will immediately raise an exception if a command returns with a non-zero value.

The slightly lower-level is Popen , which creates processes and allows fine-grained configuration of their inputs and outputs. Both check_call and check_output are implemented on top of Popen. Because of that, they share some semantics and arguments. The most important argument is shell=True, and it is most important in that it is almost always a bad idea to use it. When the argument is given, a string is expected, and is passed to the shell to parse it.

Shell parsing rules are subtle and full of corner cases. If it is a constant command, there is no benefit there: We can translate the command to separate arguments in code. If it includes some input, it is almost impossible to reliably escape it in a way that makes it impossible to introduce an injection problem. On the other hand, without this, creating commands on the fly is reliable, even in the face of potentially hostile inputs.

The following, for example, will add a user to the docker group.

```
subprocess.check_call(["usermod", "-G", "docker", "some-user"])
```

Using check_call means that if the command fails for some reason, such as the user not existing, this will automatically raise an exception. This avoids a common failure mode, where scripts do not report accurate status.

If we want to make it into a function that takes a username, it is straightforward:

```
def add_to_docker(username):
    subprocess.check_call(["usermod", "-G", "docker", username])
```

Note that this is safe to call even if the argument contains spaces, `#`, or other characters with special meanings.

In order to tell which groups the current user is currently in, we can run groups.

```
groups = subprocess.check_output(["groups"]).split()
```

Again, this will automatically raise an exception if the command fails. If it succeeds, we get the output as a string: no need to manually read and determine end conditions.

Both of these functions get common arguments. cwd allows running a command inside of a given directory. This matters for commands that look in their current directory.

```
sha = subprocess.check_output(
          ["git", "rev-parse", "HEAD"],
          cwd="src/some-project").decode("ascii").strip()
```

This will get the current git hash of the project, assuming the project is a git directory. If it is not, git rev-parse HEAD will return non-zero and cause an exception to be raised.

Note that we had to decode the output, since subprocess.check_output, like most functions in subprocess , returns a byte string, not a Unicode string. In this case, rev-parse HEAD always returns a hexadecimal string, so we used the ascii codec. This will fail on any non-ASCII characters.

There are some circumstances under which using the high-level abstractions are impossible. For example, having to send standard input or read output in chunks is not possible with them.

Popen runs a subprocess and allows fine-grained control on the inputs and outputs. While all things are, indeed, possible, most things are not easy to do correctly. The shell pattern of writing long pipelines is both unpleasant to implement; even more unpleasant to make sure there are no lingering deadlock conditions; and, most of all, unnecessary.

If a short message into standard input is needed, the best way is to use the method communicate.

```
proc = Popen(["docker", "login", "--password-stdin"], stdin=PIPE)
out, err = proc.communicate(my_password + "\n")
```

If longer input is needed, having the communicate buffer it all in memory might be problematic. While it is possible to write to the process in chunks, doing it without potentially getting deadlocks is nontrivial. The best option is often to use a temporary file:

```
with tempfile.TemporaryFile() as fp:
    fp.write(contents)
    fp.write(of)
    fp.write(email)
    fp.flush()
    fp.seek(0)
    proc = Popen(["sendmail"], stdin=fp)
    result = proc.poll()
```

In fact, in this case, we can even use the check_call function:

```
with tempfile.TemporaryFile() as fp:
    fp.write(contents)
    fp.write(of)
    fp.write(email)
    fp.flush()
    fp.seek(0)
    check_call(["sendmail"], stdin=fp)
```

If you are used to running processes in shell, you are probably used to long pipelines:

```
$ ls -l | sort | head -3 | awk '{print $3}'
```

As noted above, it is a best practice in Python to avoid true command parallelism: in all of the cases, we tried to finish one stage before reading from the next. In Python, in general, using subprocess is only used for calling out to external commands. For preprocessing of inputs, and post-processing of outputs, we usually use Python’s built-in processing abilities: in the case above, we would use sorted slices and string manipulation to simulate the logic.

The commands for text and number processing are seldom useful in Python, which has a good in-memory model for doing such processing. The general case for calling commands in scripts is for things that manipulate data in a way that is either only documented as accessible by commands – for example, querying processes via ps -ef, or where the alternative to the command is a subtle library, sometimes requiring binary binding, such as in the case of docker or git.

This is one place where translating shell scripts into Python must be done with care and thought. Where the original had a long pipeline that depended on ad hoc string manipulation via awk or sed, Python code can be less parallel and more obvious. It is important to note that in those cases, there is something lost in translation: the original low-memory requirements and transparent parallelism. However, in return we get more maintainable and more debuggable code.

## 4.3 Networking

Python has plenty of networking support . It has it from the lowest level: support of the socket-based system calls to high-level protocol supports. Some of the best approaches for problems are with built-in libraries. For other problems, the best solution is a third-party library.

The most straightforward translation of low-level networking APIs is in the socket module. This module exposes the socket object.

The HTTP protocol is simple enough so we can implement a simple client straight from the Python interactive command prompt.

```
>>> import socket, json, pprint
>>> s = socket.socket()
>>> s.connect(('httpbin.org', 80))
>>> s.send(b'GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n')
40
>>> res = s.recv(1024)
>>> pprint.pprint(json.loads(
...               res.decode('ascii').split('\r\n\r\n', 1)[1]))
{'args': {},
 'headers': {'Connection': 'close', 'Host': 'httpbin.org'},
 'origin': '73.162.254.113',
 'url': 'http://httpbin.org/get'}
```

The line s = socket.socket() creates a new socket object. There are various things that we can do with socket objects. One of them is to connect them to an endpoint: in this case, to the server httpbin.org , port 80. The default socket type is a stream, internet type: this is the way UNIX refers to TCP sockets.

After the socket is connected, we can send bytes to it. Note – on sockets, only byte strings can be sent. We read back the result and do some ad hoc HTTP response parsing – and parse the actual content as JSON.

While in general, it is better to use a real HTTP client, this showcases how to write low-level socket code. This can be useful, for example, if we want to diagnose a problem by replaying exact messages.

The socket API is subtle, and the above example has a few incorrect assumptions in it. In most cases, this code will work but will fail in strange ways in the face of corner cases.

The send method is allowed to not send all the data, if not all of it can fit into the internal kernel-level send buffer. This means that it can do a “partial send.” It returned 40, above, which was the entire length of the byte string. Correct code would have checked for the return value and send the remaining chunks until nothing is left. Luckily, Python already has a method to do it: sendall.

However, a more subtle problem occurs with recv. It will return as much as the kernel-level buffer has, because it does not know how much the other side intended to send. Again, much of the time, especially for short messages, this will work fine. For protocols like HTTP 1.0, the correct behavior is to read until the connection is closed.

Here is a fixed version of the code:

```
>>> import socket, json, pprint
>>> s = socket.socket()
>>> s.connect(('httpbin.org', 80))
>>> s.sendall(b'GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n')
>>> resp = b''
>>> while True:
...    more = s.recv(1024)
...    if more == b'':
...            break
...    resp += more
...
>>> pprint.pprint(json.loads(resp.decode('ascii').split('\r\n\r\n')[1]))
{'args': {},
 'headers': {'Connection': 'close', 'Host': 'httpbin.org'},
 'origin': '73.162.254.113',
 'url': 'http://httpbin.org/get'}
```

This is a common problem in networking code, and one that can happen using higher-level abstractions as well. Things can appear to work in simple cases while failing to work in more extreme circumstances, such as high-load or network congestion.

There are ways to test for these things. One of them is using proxies that exhibit extreme behaviors. Writing, or customizing those, will require low-level network coding using socket.

Python also has higher-level abstractions for networking. While the urllib and urllib2 modules are part of the standard library, best practices on the web evolve fast, and in general, for higher-level abstractions, third-party libraries are usually better.

One of the most popular is a third-party library, requests. With requests, getting a simple HTTP page is much simpler.

```
>>> import requests, pprint
>>> res=requests.get('http://httpbin.org/get')
>>> pprint.pprint(res.json())
{'args': {},
 'headers': {'Accept': '∗/∗',
             'Accept-Encoding': 'gzip, deflate',
             'Connection': 'close',
             'Host': 'httpbin.org',
             'User-Agent': 'python-requests/2.19.1'},
 'origin': '73.162.254.113',
 'url': 'http://httpbin.org/get'}
```

Instead of crafting our own HTTP requests out of raw bytes, all we needed to do was to give a URL, similar to a URL we might type into a browser. Requests parsed it to find the host to connect to ( httpbin.org ) the port (80, the default for HTTP) and the path (/get). Once the response came in, it automatically parsed it into headers and content, and it allowed us to access the content directly as JSON.

As easy as requests is to use, however, it is almost better to put in a little bit more effort and use the Session object. Otherwise, the default session is used. This leads to code with nonlocal side effects: one sub-library that calls requests changes some session state, which leads to another sub-library’s calls to act differently. For example, HTTP cookies are shared across a session.

The code above would be better written as:

```
>>> import requests, pprint
>>> session = requests.Session()
>>> res = session.get('http://httpbin.org/get')
>>> pprint.pprint(res.json())
{'args': {},
 'headers': {'Accept': '∗/∗',
             'Accept-Encoding': 'gzip, deflate',
             'Connection': 'close',
             'Host': 'httpbin.org',
             'User-Agent': 'python-requests/2.19.1'},
 'origin': '73.162.254.113',
 'url': 'http://httpbin.org/get'}
```

In this example, the request is simple and session state does not matter. However, this is a good habit to get into: even in the interactive interpreter, to avoid using the get, put, and other functions directly and using only the session interface.

It is natural to use an interactive environment to prototype code, which would later make it into a production program. By keeping good habits like this, we ease the transition.
