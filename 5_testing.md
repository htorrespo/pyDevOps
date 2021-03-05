<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_5
-->

# Testing

It is too often the case that code used for automating systems does not have the same attention for testing as application code. DevOps teams are often small and under tight deadlines. Such code is also hard to test, since it is meant to automate large systems, and proper isolation for testing is nontrivial.

However, testing is one of the best ways to increase code quality. It helps make code more maintainable in many ways. It also lowers defect rates. For code where defects can often mean total system outage, since it often touches all the parts of the system, this is important.

## 5.1 Unit Testing

Unit tests serve several distinct purposes. It is important to keep these purposes in mind, as the resulting pressures on the unit tests are sometimes at odds.

The first purpose is as an API usage example. This is sometimes summarized with the somewhat-inaccurate term “test driven development,” and sometimes summarized with another other somewhat-inaccurate term, “the unit tests are the documentation.”

Test driven development means writing the unit tests before the logic, but it usually has little effect on the final source code commit, which contains both the unit tests and the logic, unless care is taken to preserve the original branch-wise commit history.

However, what does show up in the commit is the “unit tests as ways to exercise the API.” It is, ideally, not the only documentation of the API. However, it does serve as a useful reference of last resort: at the very least, we know that the unit tests are calling the API correctly and get back the results they expect.

Another reason is to gain confidence that the logic expressed in the code does the right thing. Again, this is often referred to with the misnomer “regression tests,” after the most common such test: a test to make sure that a bug, detected by someone, is truly fixed. However, since the developer of the code is aware of the potential edge cases and trickier flows, they are often in a position to add that test before such a bug makes it out into the externally-observed code change: however, such a confidence-increasing test looks exactly like a “regression test.”

A final reason is to avoid incorrect future changes. This is different from the “regression test,” above, in that often the case being tested is straightforward for the code as is, and the flows involved are already covered by other tests. However, it seems like some potential optimizations or other natural changes might break this case, so including it helps a future maintenance programmer.

When writing a test, it is important to think about which of those goals it is meant to accomplish.
A good test will accomplish more than one. All tests have two potential impacts:

- Make the code better by helping future maintenance work.

- Make the code worse by making future maintenance work harder.

Every test will do some of both. A good test does more of the first, and a bad test does more of the second. One way to reduce the bad impact is to consider the question: “Is this test testing something that the code promises to do?” If the answer is “no,” this means it is valid to change the code in some way that will break the test, but will not cause any bugs. This means the test has to be changed or discarded.

When writing tests, as much as possible, it is important to test the actual contract of the code.

Here is an example:

```python
def write_numbers(fout):
    fout.write("1\n")
    fout.write("2\n")
    fout.write("3\n")
```

This function writes a few numbers into a file.

A bad test might look like this:

```python
class DummyFile:
    
    def __init__(self):
        self.written = []
    
    def write(self, thing):
        self.written.append(thing)

def test_write_numbers():
    fout = DummyFile()
    write_numbers(fout)
    assert_that(fout.written, is_(["1\n", "2\n", "3\n"]))
```

The reason this is a bad test is because it checks for a promise that write_numbers never made: that each write only writes one line.

A future refactor might look like this:

```python
def write_numbers(fout):
    fout.write("1\n2\n3\n")
```

This would keep the code correct – all users of write_numbers would still have correct files – but would cause a change in the test.

A slightly more sophisticated approach would be to concatenate the strings written.

```python
class DummyFile:
    
    def __init__(self):
        self.written = []
    
    def write(self, thing):
        self.written.append(thing)

def test_write_numbers():
    fout = DummyFile()
    write_numbers(fout)
    assert_that("".join(fout.written), is_("1\n2\n3\n"))
```

Note that this test would work both before and after the hypothetical “optimization” that we suggested above. However, this still tests more than the implied contract of write_numbers. After all, the function is supposed to operate on files: it might use another method to write.

The test above would break if we modified write_numbers to:

```python
def write_numbers(fout):
    fout.writelines(["1\n",
                     "2\n",
                     "3\n"]
```

A good test is one that would only break if there was a bug in the code. However, this code still works for the users of write_numbers , meaning that the maintenance now involved unbreaking a test, pure overhead.

Since the contract is to be able to write to file objects, it is best to supply a file object. In this case, Python has one ready-made:

```python
def test_write_numbers():
    fout = io.StringIO()
    write_numbers(fout)
    assert_that(fout.getvalue(), is_("1\n2\n3\n"))
```

In some cases, this will require writing a custom fake. We will cover the concept of fakes, and how to write them, later.

We talked about the implicit contract of write_numbers . Since it had no documentation, we could not know what the original programmer’s intent was. This is, unfortunately, common – especially in internal code, only used by other pieces of the project. Of course, it is better to clearly document programmer intent. In the face of lack of clear documentation, however, it is important to make reasonable assumptions on the implicit contract.

Above, we used the functions assert_that and is_ to verify that the values were what we expected. Those functions come from the hamcrest library. This library, ported from Java, allows specifying properties of structures and checks that they are satisfied.

When using the pytest test runner to run unit tests, it is possible to use regular Python operators with the assert keyword and get useful test failures. However, this binds the tests to a specific runner, as well as having a specific set of assertions that get treated especially for useful error messages.

Hamcrest is an open-ended library: while it has built-in assertions for the usual things (equality, comparisons, sequence operations, and more), it also allows defining specific assertions. Those come in handy when handing complicated data structures, such as those returned from APIs, or when only specific assertions can be guaranteed by the contract (for example, the first three characters can be arbitrary but must be repeated somewhere inside of the rest of the string).

This allows to test the exact contract of the function. In particular, this is another tool in avoiding testing “too much”: testing implementation details that can change, requiring changing the test when no real users have been broken. This is crucial for three reasons.

One is straightforward: time spent updating tests that could have been avoided is time wasted. DevOps teams are usually small, and there is little room to waste resources.

The second is that getting used to changing tests when they fail is a bad habit. It means when behavior has changed as a result of a bug , people will assume that the right thing to do is to update the test.

Finally, and most importantly, the combination of those two will lower the return on investment on unit testing, and even worse, the perceived return on investment. As a result, there will be organizational pressure to spend less time writing tests. Bad tests that test implementation details are the single biggest cause for the meme that “it is not worth it to write unit tests for DevOps code.”

As an example, let’s assume we have a function where all we can assert confidently is that the result has to be divisible by one of the arguments.

```python
class DivisibleBy(hamcrest.core.base_matcher.BaseMatcher):
    
    def __init__(self, factor):
        self.factor = factor
    
    def _matches(self, item):
        return (item % self.factor) == 0
    
    def describe_to(self, description):
        description.append_text('number divisible by')
        description.append_text(repr(self.factor))

def divisible_by(num):
    return DivisibleBy(num)
```

By convention, we wrap constructors in a function. This is usually useful if we want to convert the argument to a matcher, which in this case would not make sense.

```python
def test_scale():
    result = scale_one(3, 7)
    assert_that(result,
                any_of(divisible_by(3),
                       divisible_by(7)))
```

We would get an error like the following:

```python
Expected: (number divisible by 3 or number divisible by 7)
     but: was <17>
```

It lets us test exactly what the contract of scale_one promises: in this case, that it would scale up one of the arguments by an integer factor.

The emphasis on the importance of testing precise contracts is not accidental. This emphasis, which is a skill that is possible to learn and has principles that are possible to teach, makes unit tests into something that accelerate the process of writing code rather than make it slower.

Much of the reason people have aversion to unit tests as “something that wastes time for DevOps engineers” and leads to a lot of poorly tested code that is foundational for business processes such as deployment of software is this misconception. Properly applying principles of high-quality unit testing leads to a more reliable foundation for operational code.

## 5.2 Mocks, Stubs, and Fakes

Typical DevOps code has outsized effects on the operating environment. Indeed, this is almost the definition of good DevOps code: it replaces a significant amount of manual work. This means that testing DevOps code needs to be done carefully: we cannot simply spin up a few hundred virtual machines for each test run.

Automating operations means writing code that, run haphazardly, can have significant impact on production systems. When testing the code, it is worthwhile to have as few of these side effects as possible. Even if we have high-quality staging systems, sacrificing one every time there is a bug in operational code would lead to a lot of wasted time. It is important to remember that unit tests run on the worst code produced: the act of running them, and fixing bugs, means even code committed into feature branches is likelier to be in better condition.

Because of that, we often try to run unit tests against a “fake” system. Classifying what we mean by “fake,” and how it impacts both unit tests and code design, is important: it is worthwhile thinking about how to test the code well before starting to write it.

The neutral term for things that substitute for the systems not under test is “test doubles.” Fakes, mocks , and stubs usually have more precise meaning, though in casual conversation they will be used interchangeably.

The most “authentic” test double is a “verified fake.” A verified fake fully implements the interface of the system not under test, though often simplified: perhaps less efficiently implemented, often without touching any external operating system. The “verified” refers to the fact that the fake has its own tests, verifying that it does indeed implement the interface.

An example of a verified fake is using a memory-only SQLite database instead of a file-based one in tests. Since SQLite has its own tests, this is a verified fake: we can be confident it behaves like a real SQLite database.

Below the verified fake is the fake. The fake implements an interface but often does it in such a rudimentary form that the implementation is simple, and not worth the effort to test.

For example, it is possible to create an object with the same interface as subprocess.Popen but that never actually runs the process: instead, it simulates a process that consumes all standard input, and outputs some predetermined content into standard output and exits with a predetermined code.

This object, if simple enough, might be a stub . A stub is a simple object that answers with predetermined data, always the same, holding almost no logic. This makes it easier to write, but it does make it constrained in what tests it can do.

An inspector, or a spy, is an object that attaches to a test double and monitors the calls. Often, part of the contract of a function is that it will call some method with specific values. An inspector records the calls and can be used in assertions to make sure the right calls got the right arguments.

If we combine an inspect or with a stub or a fake, we get a mock . Since this means that the stub/fake will have more functionality than the original (at least, whatever is needed to check the recording), this can lead to some side effects. However, the simplicity and immediacy of creating mocks often compensates by making testing code simpler.

## 5.3 Testing Files

The filesystem is, in many ways, the most important thing about a UNIX system. While the slogan “everything is a file” falls short of describing modern systems, the filesystem is still at the heart of most operations.

The filesystem has several properties that are worthwhile to consider when thinking about testing file-manipulation code.

First, filesystems tend to be robust. While bugs in filesystems are not unknown, they are rare, far between, and usually only triggered by extreme conditions or an unlikely combination of conditions.

Next, filesystems tend to be fast. Consider the fact that unpacking a source tarball, a routine operation, will create many small files (on the order of several kilobytes) in quick succession. This is a combination of fast system-call mechanisms combined with sophisticated cache semantics when reading or writing files.

Filesystems also have a curious fractal property: with the exception of some esoteric operations, a sub-sub-sub-directory supports the same semantics as the root directory.

Finally, filesystems have a very thick interface. Some of it will be built into Python, even – consider that the module system reads files directly. There are also third-party C libraries that will use their own internal wrappers to access the filesystem as well as several ways to open files even in Python: the built-in file object as well as the os.open low-level operations.

This combines to the following conclusion: for most file-manipulation code, faking out or mocking the filesystem is a low return on investment. The investment, in order to make sure we are only testing the contract of a function, is considerable; since the function could, conceivably, switch to low-level file-manipulation operations, we would need to reimplement a significant portion of Unix file semantics. The return is low; using the filesystem directly is fast, reliable, and, as long as the code merely allows us to pass an alternative “root path,” almost side-effect free.

The best way to design file-manipulation code is to allow passing in such a “root path” argument, even if the default is /. Given such design, the best way to test is to create a temporary directory, populate it appropriately, call the code, and then garbage collect the directory.

If we create the temporary directory using Python’s built-in tempfile module , then we can configure the Tox runner to put the temporary file inside of Tox’s built-in temporary directory, thus keeping the general file system clean and, usually, being compatible with whatever version control ignore file already ignores Tox artifacts.

```python
setenv =
    TMPDIR = {envtmpdir}
commands =
    python -m 'import os;os.makedirs(sys.argv[1])' {envtmpdir}
    # rest of test commands
```

Creating the temporary directory is important, since Python’s tempfile will only use the environment variable if pointing to a real directory.

As an example, we will write tests for a function that looks for .js files and renames them to .py.

```python
def javascript_to_python_1(dirname):
    for fname in os.listdir(dirname):
        if fname.endswith('.js'):
            os.rename(fname, fname[:3] + '.py')
```

This function uses the os.listdir call to find the file names and then renames them with os.rename.

```python
def javascript_to_python_2(dirname):
    for fname in glob.glob(os.path.join(dirname, "∗.js")):
        os.rename(fname, fname[:3] + '.py')
```

This function uses the glob.glob function to filter by wildcard all the files that match the `*.js` pattern.

```python
def javascript_to_python_3(dirname):
    for path in pathlib.Path(dirname).iterdir():
        if path.suffix == '.js':
            path.rename(path.parent.joinpath(path.stem + '.py'))
```

The function uses the built-in module pathlib (new in Python 3) to iterate on the directory and find its children.

The real function under test is not sure which implementation to use:

```python
def javascript_to_python(dirname):
    return random.choice([javascript_to_python_1,
                          javascript_to_python_2,
                          javascript_to_python_3])(dirname)
```

Since we cannot be sure which implementation the function will use, we are left with only one choice: test the actual contract.

In order to write a test, we will define some helper code. This code, in a real project, will live in a dedicated module, possibly named something like helpers_for_tests. This module would be tested, with its own unit tests.

We first create a context manager for our temporary directory. This will ensure, as much as ensuring is possible, that the temporary directory will be cleaned up.

```python
@contextlib.contextmanager
def get_temp_dir():
    temp_dir = tempfile.mkdtemp()
    try:
        yield temp_dir
    finally:
        shutil.rmtree(temp_dir)
```

Since this test needs to create a lot of files, and we do not care about their contents too much, we define a helper method for that.

```python
def touch(fname, content=''):
    with open(fname, 'a') as fpin:
        fpin.write(content)
```

Now with the help of these functions, we can finally write a test:

```python
def test_javascript_to_python_simple():
    with get_temp_dir() as temp_dir:
        touch(os.path.join(temp_dir, 'foo.js'))
        touch(os.path.join(temp_dir, 'bar.py'))
        touch(os.path.join(temp_dir, 'baz.txt'))
        javascript_to_python(temp_dir)
        assert_that(set(os.listdir(temp_dir)),
                    is_({'foo.py', 'bar.py', 'baz.txt'}))
```

For a real project, we would write more tests, many of them possibly using our get_temp_dir and touch helpers above.

If we have a function that is supposed to check a specific path, we can have it take an argument to “relativize” its paths.

For example, let us say we want a function to analyze our Debian installation paths and give us a list of all domains that we download packages from.

```python
def _analyze_debian_paths_from_file(fpin):
    for line in fpin:
        line = line.strip()
        if not line:
            continue
        line = line.split('#', 1)[0]
        parts = line.split()
        if parts[0] != 'deb':
            continue
        if parts[1][0] == '[':
            del parts[1]
        parsed = hyperlink.URL.from_text(parts[1].decode('ascii'))
        yield parsed.host
```

A naive approach would be to test _analyze_debian_paths_from_file. However, it is an internal function and has no contract. The implementation can change, perhaps reading the files and then scanning all strings, or possibly breaking up this function and letting the top-level handle the line loop.

Instead, we want to test the public API:

```python
def analyze_debian_paths():
    for fname in os.listdir('/etc/apt/sources.list.d'):
        with open(os.path.join('/etc/apt/sources.list.d', fname)) as fpin:
            yield from _analyze_debian_paths_from_file(fpin)
```

However, we cannot control the directory /etc/apt/sources.list.d without root privileges, and even with root privileges, this would be a risk: letting each test run control such a sensitive directory. Additionally, many Continuous Integration systems are not designed for running tests with root privileges, for good reasons, making this a problematic approach.

Instead, we can generalize the function a little bit. This means intentionally expanding the official, public API of the function to allow testing. This is definitely a trade-off.

However, the expansion is minimal: all we need is an explicit directory in which to work. In return, we get to simplify our testing requirements while avoiding any kind of “patching,” which inevitably starts poking at private implementation details.

```python
def analyze_debian_paths(relative_to='/'):
    sources_dir = os.path.join(relative_to, 'etc/apt/sources.list.d')
    for fname in os.listdir(sources_dir):
        with open(os.path.join(sources_dir, fname)) as fpin:
            yield from _analyze_debian_paths_from_file(fpin)
```

Now, using the same helpers as before, we can write a simple test for this:

```python
def test_analyze_debian_paths():
    with get_temp_dir() as root:
        touch(os.path.join(root, 'foo.list'),
              content='deb http://foo.example.com\n')
        ret = list(analyze_debian_paths(relative_to=root))
        assert(ret, equals_to(['foo.example.com']))
```

Again, in a real project, we would write more than one test and try to make sure many more cases are covered. Those could be built using the same techniques.

It is a good habit to add a relative_to parameter to any function that accesses specific paths.

## 5.4 Testing Processes

Testing process-manipulation code is often a subtle endeavor, full of trade-offs. In theory, process running code has a thick interface with the operating system; we covered the subprocess module, but it is possible to use the os.spawn* functions directly, or even use code os.fork and os.exec* functions. Likewise, the standard output/input communication mechanism can be implemented in many ways, including using the Popen abstraction or directly manipulating file descriptors with os.pipe and os.dup.

Process-manipulation code can also be some of the most fragile. Running external commands depends on the behavior of those commands, as a starting point. The inter-process communication means that the flow is inherently concurrent. It is too easy to make the mistake of making the tests rely on ordering assumptions that are not always true. Those mistakes can lead to “flaky” tests: ones that pass most of the time, but fail under seemingly random circumstances.

Those ordering assumptions can sometimes be true more often on development machines, or unloaded machines, which means bugs will only be exposed in production, or possibly in production only in extreme circumstances.

This is one of the reasons the chapter about using processes concentrated on ways to reduce concurrency and have things more sequential. For this reason, too, it is worthwhile, carefully designing process code to be reliably testable. That design, in itself, will often cause pressure on the code to be simple and reliable.

If the code just uses subprocess.check_call and subprocess.check_output, without taking advantage of exotic parameters, we can often use a simplified form of a pattern called “dependency injection” to make it testable. In this case, “dependency injection” is just a fancy way of saying “passing parameters to a function.”

Consider the following function:

```python
def error_lines(container_name):
    logs = subprocess.check_output(["docker", "logs", container_name])
    for line in logs:
        if 'error' in line:
            return line
```
This function is unpleasant to test. We can use advanced patching to replace subprocess.check_output, but this would be error prone and rely on implementation details. Instead, we can explicitly elevate that implementation detail into being a part of the contract:

```python
def error_lines(container_name, runner=subprocess.check_output):
    logs = runner(["docker", "logs", container_name])
    
    for line in logs:
        if 'error' in line:
            yield line.strip()
```
Now that runner is part of the official interface, testing becomes much easier. This might seem a trivial change, but it is deeper than it looks; in some sense, error_lines has now voluntarily constrained its interface to process running.

We might want to test it with something like the following:

```python
def test_error_lines():
    
    container_name = 'foo'
    
    def runner(args):
        
        if args[0] != 'docker':
            raise ValueError("Can only run docker", args)
        
        if args[1] != 'logs':
            raise ValueError("Can only run docker logs", args)
        
        if args[2] != container_name:
            raise ValueError("No such container", args[2])
        
        return iter(["hello\n", "error: 5 is not 6\n", "goodbye\n"])
    
    ret = error_lines(container_name, runner=runner)
    assert_that(list(ret), is_(["error: 5 is not 6"))
```

Note that, in this case, we did not restrict ourselves to only checking the contract: error_lines could have run, for example, docker logs -- <container_name>. However, one advantage of our method is that we can slowly improve our fidelity and only improve the test.

For example, we can add to runner:

```python
def runner(args):
    
    if args[0] != 'docker':
        raise ValueError("Can only run docker", args)
    
    if args[1] != 'logs':
        raise ValueError("Can only run docker logs", args)
    
    if args[2] == '--':
        arg_container_name = args[3]
    
    else:
        arg_container_name = args[2]
    
    if args_container_name != container_name:
        raise ValueError("No such container", args[2])
    
    return iter(["hello\n", "error: 5 is not 6\n", "goodbye\n"])
```

This will still work with the old version of the code and will also work with post-modification code. Fully emulating the docker is not realistic or worthwhile. However, this approach would slowly improve the accuracy of the test, with no downsides.

If a significant amount of our code interfaces, for example, with docker, we can eventually factor out a mini-docker-emulator like that into its own test helper library.

Using higher-level abstractions for process running helps with this sort of approach. The seashore library, for example, separates the part that calculates the commands from the low-level runner, which allows substituting only the low-level one.

```python
def error_lines(container_name, executor):
    logs, _ignored = executor.docker.logs(container_name).batch()
    
    for line in logs.splitlines():
        if 'error' in line:
            yield line.strip()
```
When run in production, somewhere at the top, an executor object will be created with code that looks like this:
executor = seashore.Executor(seashore.Shell())

That object will be passed down to whatever is calling error_lines and used there. In general, when using seashore , we leave the creation of the executor to the top-level functionality.

In the test, we create our own shell:

```python
@attr.s
class DummyShell:
    
    _container_name = attr.ib()
    
    def batch(self, ∗args, ∗∗kwargs):
        if (args == ['docker', 'logs', self._container_name] and
            kwargs == {}):
            return "hello\nerror: 5 is not 6\ngoodbye\n", ""
        raise ValueError("unknown command", self, args, kwargs)

def test_error_lines():
    container_name = 'foo'
    executor = seashore.Executor(DummyShell(container_name))
    ret = error_lines(container_name, executor)
    assert_that(list(ret), is_(["error: 5 is not 6"]))
```

Using the attrs library, especially when writing various fakes, is often a good idea. Fakes tend to be, intentionally, simple objects. Since they will be involved in assertions and exceptions, it is useful to have high-quality representations of them. This is exactly the kind of boilerplate that attrs helps reduce.

Again, we might need to slowly upgrade our fidelity.

Because processes are so hard to test, it is good to use process running only when necessary. Especially when porting over shell scripts to Python – often a good idea when they grow in complexity – it is good to substitute long pipelines with in-memory data processing.

Especially if we factor the code the right way, with the data processing as a simple pure function that takes an argument and returns a value, the bulk of the code becomes a pleasure to test.

Imagine, for example, the pipeline,

```terminal
ps aux | grep conky | grep -v grep | awk '{print $2}' | xargs kill
```
This will kill all processes that have conky in their names.

Here is a way to refactor the code to make it easier to test:

```python
def get_pids(lines):
    for line in lines:
        if 'conky' not in line:
            continue
        parts = line.split()
        pid_part = parts[1]
        pid = int(pid_part)
        yield pid

def ps_aux(runner=subprocess.check_output):
    return runner(["ps", "aux"])

def kill(pids, killer=os.kill):
    for pid in pids:
        killer(pid, signal.SIGTERM)

def main():
    kill(get_pid(ps_aux()))
```

Note how the most complicated code is now in a pure function: get_pids . Hopefully, this means most bugs will be there, and we can unit test against them.

The code that is harder to unit test, get_pids, where we have to do ad hoc dependency injection, is now in simple functions that have fewer failure modes.

The main logic is in functions that do data processing. Testing those just requires supplying simple data structure and observing the return value. Moving potential bugs from the system-related code, which requires more effort to unit test, to the pure logic, which is easier to unit test, means reducing the bugs; more bugs will be caught with unit tests.

## 5.5 Testing Networking

In the requests library documentation, using the Session object falls under the “advanced” section. This is unfortunate. For anything other than throwaway scripts, or interactive REPL usage, using the Session object is the best option. Testing is by far the least of the reasons – but once Session is used, testing becomes a lot easier.

Simple example code using requests might look like this:

```python
def get_files(gist_id):
    gist = requests.get(f"https://api.github.com/gists/{gist_id}").json()
    result = {}
    for name, details in gist["files"].items():
        result[name] = requests.get(details["raw_url"]).content
    return result
```

This would be hard to test in isolation. Instead, we rewrite it to take an explicit session object:

```python
def get_files(gist_id, session=None):
    if session is None:
        session = requests.Session()
    gist = session.get(f"https://api.github.com/gists/{gist_id}").json()
    result = {}
    for name, details in gist["files"].items():
        result[name] = session.get(details["raw_url"]).content
    return result
```

The code is almost identical. However, now testing becomes a simple matter of writing an object with a get method .

```python
@attr.s(frozen=True)
class Gist:
    files = attr.ib()

@attr.s(frozen=True):
class Response:
    
    content = attr.ib()
    
    def json(self):
        return json.loads(content)

@attr.s(frozen=True)
class FakeSession:
    
    _gists = attr.ib()
    
    def get(self, url):
        parsed = hyperlink.URL.from_text(url)
        if parsed.host == 'api.github.com':
            tail = path.rsplit('/', 1)[-1]
            gist = self._gists[tail]
            res = dict(files={name: f'http://example.com/{tail}/{name}'
                              for name in gist.files})
            return Repsonse(json.dumps(res))
        if parsed.host == 'example.com':
            _ignored, gist, name = path.split('/')
            return Response(self.gists[gist][name])
```

This is a bit long-winded. We can sometimes, if this functionality is localized and writing a whole helper library is not worth it, use the unittest.mock library .

```python
def make_mock():
    gist_name = 'some_name'
    files = {'some_file': 'some_content'}
    session = mock.Mock()
    session.get.content.return_value = 'some_content'
    session.get.json.return_value = json.dumps({'files': 'some_file'})
    return session
```

This is a “quick and dirty” hack, counting on the fact (that is not in the contract) that the file content is retrieved using content, and the gist’s logical structure is retrieved using json. However, it is often better to write a quick test using mocks that depend a little on the implementation details rather than not writing a test at all.

It is important to think of tests like this as “technical debt” and improve them at some point to depend more on the contract and less on the implementation details. A good way to do it is to put a comment in the code, and link it to an issue tracker. This also makes it obvious to test code readers that this is still a work in progress.

The other important thing is that, if a new implementation breaks the test, the right way to fix it is, in general, not to write another test against the new implementation. The right way to fix it is to move more of the test to contract-based testing. This can be done by first improving the test, but making sure it runs against the old code. Then comes refactoring the code and seeing the test still passing.

When writing network code that deals with lower-level concepts, such as sockets, similar ideas still apply. Since the creation of the socket object is separate from any usage of it, a lot of mileage can be gotten out of writing functions that accept socket objects, and creating them outside.

In order to simulate extreme conditions and see if our code can work in spite of them, we might want to use something like the following as a socket fake:

```python
@attr.s
class FakeSimpleSocket:
    
    _chunk_size = attr.ib()
    _received = attr.ib(init=False, factory=list)
    _to_send = attr.ib()
    
    def connect(self, addr):
        pass
    
    def send(self, blob):
        actually_sent = blob[:chunk_size]
        self._received.append(actually_sent)
        return len(actually_sent)
    
    def recv(self, max_size):
        chunk_size = min(max_size, self._chunk_size)
        received, self._to_send = (self._to_send[:chunk_size],
                                   self._to_send[chunk_size:])
        return received
```

This allows us to control the size of “chunks.” An extreme test would be to use a chunk_size of 1. This means bytes would go out one at a time, and they would be received one at a time. No real network would be this bad, but a unit test allows us to simulate more extreme conditions than any reasonable network.

This fake is useful to test networking code. For example, this code does some ad hoc HTTP to get a result:

```python
def get_get(sock):
    sock.connect(('httpbin.org', 80))
    sock.send(b'GET /get HTTP/1.0\r\nHost: httpbin.org\r\n\r\n')
    res = sock.recv(1024)
    return json.loads(res.decode('ascii').split('\r\n\r\n', 1)[1]))
```

It has a subtle bug in it. We can uncover the bug with a simple unit test, using the socket fake.

```python
def test_get_get():
    result = dict(url='http://httpbin.org/get')
    headers = 'HTTP/1.0 200 OK\r\nContent-Type: application/json\r\n\r\n'
    output = headers + json.dumps(result)
    fake_sock = FakeSimpleSocket(to_send=output, chunk_size=1)
    value = get_get(fake_sock)
    assert_that(value, is_(result))
```

This test would fail: our get_get assumes a good quality network connection, and this simulates a bad one. It would succeed if we changed chunk_size to 1024.

We could run the test in a loop, testing chunk sizes from 1 to 1024. In a real test we would also check the sent data, and possibly also send invalid results to see the response. The important thing, however, is that none of those things need setting up clients or servers, or trying to realistically simulate bad networks.
