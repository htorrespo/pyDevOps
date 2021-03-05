<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_6
-->
# 6 Text Manipulation

Automation of UNIX-based systems often involves text manipulation. Many programs are configured with textual configuration files. Text is the output format, and the input format, of many systems. While tools like sed, grep, and awk have their place, Python is a powerful tool for sophisticated text manipulation.

## 6.1 Bytes, Strings, and Unicode

When manipulating text or text-like streams, it is easy to write code that fails in funny ways when encountering a foreign name, or emoji. These are no longer mere theoretical concerns: you will have users from the entire world, who insist on their usernames reflecting how they spell their names. You will have people who write git commits with emojis in them. In order to make sure to write robust code, which does not fail in ways that, to be fair, seem a lot less funny when they case a 3 a.m. page, it is important to understand that “text” is a subtle thing.

You can understand the distinction, or you can wake up at 3 a.m. when someone tries to log in with an emoji username.

Python 3 has two distinct types that both represent the kind of things that are often in UNIX “text” files: bytes and strings. Bytes correspond to what RFCs usually refer to as an “octet-stream.” This is a sequence of values that fit into 8 bits, or in other words, a sequence of numbers that are in the range 0 to 256 (including 0 and not including 256). When all of these values are below 128, we call the sequence “ASCII” (American Standard Code of Information Interchange) and assign to the numbers the meaning ASCII has assigned them. When all of these values are between 32 and 128 (including 32 and not including 128), we call the sequence “printable ASCII,” or “ASCII text.” The first 32 characters are sometimes called “Control characters.” The “Ctrl” key on keyboards is a reference to that – its original purpose was to be able to input those characters.

ASCII only encompasses the English alphabet, used in “America.” In order to represent text in (almost) any language, we have Unicode. Unicode code points are (some of the) numbers between 0 and 2∗∗32 (including 0 and not including 2∗∗32). Each Unicode code point is assigned a meaning. Successive versions of the standards leave assigned meanings as is, but add meanings to more numbers. An example is the addition of more emojis. The International Standards Organization, ISO, ratifies versions of Unicode in its 10464 standards. For this reason, Unicode is sometimes called ISO-10464.

Unicode points that are also ASCII have the same meaning – if ASCII assigns a number “uppercase A,” then so does Unicode.

Properly speaking, only Unicode is “text.” This is what Python strings represent. Converting bytes to strings, or vice versa, is done with an encoding . The most popular encoding these days is UTF-8. Confusingly, turning the bytes to text is “decoding.” Turning the text to bytes is “encoding.”

Remembering the difference between encoding and decoding is crucial in order to manipulate textual data. A way to remember it is that since UTF-8 is an encoding, moving from strings to UTF-8 encoded data is “encoding,” while moving from UTF-8 encoded data to strings is “decoding.”

UTF-8 has an interesting property: when given a Unicode string that happens to be ASCII, it will produce bytes with the values of the code points. This means that “visually,” the encoded and decoded form will look the same.

```python
>>> "hello".encode("utf-8")
b'hello'
>>> "hello".encode("utf-16")
b'\xff\xfeh\x00e\x00l\x00l\x00o\x00'
```
