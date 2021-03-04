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
>>> with open("Untitled.xcf", "rb") as fp:
...     header = fp.read(100)

Here we open a file. The rb argument stands for “read, binary.” We read the first hundred bytes. We will need far fewer, but this is often a useful tactic. Many files have some metadata at the beginning.
>>> header[:9].decode('ascii')
'gimp xcf '

The first nine characters can actually be decoded to ASCII text, and happen to be the name of the format.
>>> header[9:9+4].decode('ascii')
'v011'

The next four characters are the version. This file is the 11th version of XCF.
>>> header[9+4]
0

A 0 byte finishes the “what is this file” metadata. This has various advantages.
>>> struct.unpack('>I', header[9+4+1:9+4+1+4])
(1920,)

The next four bytes are the width, as a number in big-endian format. The struct module knows how to parse these. The > says it is big endian, and the I says it is an unsigned 4-byte integer.
>>> struct.unpack('>I', header[9+4+1+4:9+4+1+4+4])
(1080,)

The next four bytes are the width. This simple code gave us the high-level data: it confirmed that this is XCF, it showed what version of the format it is, and we could see the dimensions of the image.

When opening files as text, the default encoding is UTF-8. One advantage of UTF-8 is that it is designed to fail quickly if something is not UTF-8: it is carefully designed to fail on ISO-8859-[1-9], which predates Unicode, as well as on most binary files. It is also backwards compatible with ASCII, which means pure ASCII files will still be valid UTF-8.

The most popular way to parse text files is line by line, and Python supports that by having an open text file be an iterator that yields the lines in order.
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

Usually we will not call next directly, but use for. Additionally, usually we use files as context managers, to make sure they close at a well-understood point. However, especially in REPL scenarios, there is a trade-off: opening the file without a context manager allows us to explore reading bits and pieces.

Files on a unix system are more than just blobs of data. They have various metadata attached, which can be queried and sometimes changed.

The rename system call is wrapped in the os.rename Python function. Since rename is atomic, this can help implement operations that require a certain state.

In general, note that the os module tends to be a thin wrapper over operating system calls. The discussion here is relevant to UNIX-like systems: Linux, BSD-based systems, and, for the most part, Mac OS X. It is worth keeping in mind, but it is not worth pointing each place where we are making UNIX-specific assumptions.

For example,
with open("important.tmp", "w") as fout:
    fout.write("The horse raced past the barn")
    fout.write("fell.\n")
os.rename("important.tmp", "important")

This ensures that when reading the important file , we do not accidentally misunderstand the sentence. If the code crashes in the middle, instead of believing that the horse raced past the barn, we get nothing from important. We only rename important.tmp to important at the end, after the last word has been written to the file.

The most important example of a file-which-is-not-a-blob, in UNIX, is a directory. The os.makedirs function allows us to ensure a directory exists easily with
os.makedirs(some_path, exists_ok=True)

This combines powerfully with the path operations from os.path to allow safe creation of a nested file:
def open_for_write(fname, mode=""):
    os.makedirs(os.path.dirname(fname), exists_ok=True)
    return open(fname, "w" + mode)
with open_for_write("some/deep/nested/name/of/file.txt") as fp:
    fp.write("hello world")

This can come in useful, for example, when mirroring an existing file layout.

The os.path module has, mostly, string manipulation functions that assume strings are file names. The dirname function returns the directory name, so os.path.dirname("a/b/c") would return a/b. Similarly, the function basename returns the “file name,” so os.path.basename("a/b/c") would return c. The inverse of both is the os.path.join function, which join paths: os.path.join("some", "long/and/winding", "path") would return some/long/and/finding/path.

Another set of functions in the os.path module has a slightly higher-level abstraction for getting file metadata. It is important to note that these functions are often light wrappers around operating system functionality, and do not try to hide operating system quirks. This means that operating system quirks can “leak” through the abstraction.

The biggest metadata is os.path.exists: does the file exist? This comes in handy sometimes, though often it is better to write code in a way that is agnostic of file existence: file existence can have races. Subtler are the os.path.is... functions: isdir, isfile, islink, and more can decide if a file name points to what we expect.

The os.path.get... functions get non-boolean metadata: access time, modification time, c-time (sometimes shortened to “creation time,” but misleadingly not the actual time of creation in a set of subtle circumstances, and more accurately refer to as “i-node modification time”), and getsize getting the size of the file.

The shutil module (“shell utilities”) contains some higher-level operations. shutil.copy will copy a file’s contents as well as metadata. shutil.copyfile will copy contents only. shutil.rmtree is the equivalent of rm -r, while shutil.copytree is the equivalent of cp -r.

Finally, temporary files are often useful. Python’s tempfile module produces temporary files that are secure and resistant to leaks. The most useful functionality is NamedTemporaryFile , which can be used as a context.

A typical usage looks like this:
with NamedTemporaryFile() as fp:
    fp.write("line 1\n")
    fp.write("line 2\n")
    fp.flush()
    function_taking_file_name(fp.name)

Note that the fp.flush there is important. The file object caches write until closed. However, NamedTemporaryFile will vanish when closed. Explicitly flushing it is important before calling a function that will reopen the file for reading.

