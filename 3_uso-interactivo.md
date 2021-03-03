<!--
https://link-springer-com.ezproxy.unal.edu.co/chapter/10.1007/978-1-4842-4433-3_3
-->

# Uso interactivo

Python is often used for exploratory programming. Often, the final result is not the program but an answer to a question. For scientists, the question might be “what is the likelihood of a medical intervention working?” For people troubleshooting computers, the question might be “which log file has the message I need?”

However, regardless of the question, Python can often be a powerful tool to answer it. More importantly, in exploratory programming, we expect to encounter more questions, based on the answer.

The interactive model in Python comes from the original Lisp environments’ “Read-Eval-Print Loop” (REPL, for short). The environment reads a Python expression, evaluates it in an environment that persists in memory, prints the result, and loops back.

The REPL environment native to Python is popular because it is built in. However, a few third-party REPL tools are even more powerful, built to do things that the native one could not or would not. These tools give a powerful way to interact with the operating system, exploring and molding until the desired state is achieved.

## 3.1 Native Console

