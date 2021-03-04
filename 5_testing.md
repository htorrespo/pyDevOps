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
