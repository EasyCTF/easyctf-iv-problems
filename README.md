OpenCTF Problem Import via Github
-------------------------------------------------

Each problem belongs in its own folder under the root folder of the repository. The name of the folder will become a readable identifier for the problem, so it should only contain letters, numbers, and `_`.

Inside this folder, there are 3 required files:

- `problem.yml`: This is probably the most important file. It contains all the metadata for the problem, including:
  - `author`: Your username on the EasyCTF website.
  - `title`: The title of the problem.
  - `category`: The category of the problem.
  - `value`: An integer value for the problem.
  - `hint`: A hint for the problem. (no markdown supported)
  - `autogen`: (`true`/`false`) whether or not the problem is autogenerated. If you put `true`, read on to see what needs to be added to the grader.
- `description.md`: A problem description. As the extension suggests, you may use markdown in this file.
- `grader.py`: A python file containing several functions that relate to grading the problem (and generating, if autogen is enabled).

## The Grader

At the most basic level, the grader should contain a `grade` function that takes in two parameters (you can call them whatever you like): (1) an instance of the random library used for autogenerating problems, and (2) the flag that the user actually typed into the input box. It should return a tuple `(correct, message)`, where `correct` is a boolean value signifying whether the user got the problem right, and `message`, a custom message to be displayed.

An example of a basic grader looks like this:

```python
def grade(random, key):
    if key.find("this_is_the_flag") != -1:
        return True, "Correct!"
    return False, "Nope."
```

If autogen is not enabled, please do not use `random` while judging the key. It's meant for checking problems that were autogenerated with the same seed. Also, note that I'm using `key.find` rather than comparing them with equals. This is useful if you want to match both `flag{this_is_the_flag}` and `this_is_the_flag`, or if you are trying to perform a case-insensitive match.

## Autogen

If autogen were enabled, you would need to add an additional function to the grader, called `generate`. Essentially, this function is required for generating the problem. It takes in a single parameter, the same instance of `random` as the `grade` function gets, and must return a dictionary of values (obviously, this dictionary can be blank, but ideally it shouldn't be).

Possible keys that you can include in your dictionary are:
- `variables`: This should be another dictionary of variables that are inserted into the description during runtime. More details on how these variables are inserted comes in a later section. You would probably want to use the `random` variable that is passed to the function to generate a value that is different per team.
- `files`: This is similar to `variables` in that it is a dictionary of keys that are inserted into the description. However, instead of returning variables directly, you should return a function that takes in the `random` variable and returns a File object (could be StringIO as well). Files generated this way are stored into the static container during runtime.

An example using both `variables` and `files` is:

```python
from cStringIO import StringIO
def gen_b(random):
    b = random.choice(range(50, 100))
    return b
def generate(random):
    a = random.choice(range(50, 100))
    return dict(variables={
        "a": a
    }, files={
        "b.txt": gen_b()
    })
```

`files` contains a dictionary mapping filenames to either `StringIO` or `BytesIO` objects. Note that we are returning `gen_b` **with** `()`. This way, the platform can call this function at runtime, generating a different problem for every user.

**This part is very important**. Filenames usually have extensions, like `ciphertext.txt`. In this case, we are passing the filename into the description as a string. But Python template strings don't allow symbols such as periods. Therefore, ALL FUNCTION NAMES ARE SANITIZED by replacing anything matching this regex: `[^a-zA-Z]+` with `_` (underscore). This way, `ciphertext.txt` becomes `ciphertext_txt` in the description. Here is a description that uses the above generator.

```markdown
Help me decipher [this](${b_txt}).
```

Here is a full example using autogen:

problem.yml:

```yaml
title: Caesar Cipher
value: 20
author: michael
autogen: true
```

description.md:

```markdown
Help me decipher [this ciphertext](${ciphertext_txt}).
```

grader.py:

```python
from cStringIO import StringIO
from string import maketrans

flag = "caesar_cipher_is_fun!"
alphabet = "abcdefghijklmnopqrstuvwxyz"

def get_problem(random):
    n = random.randint(1, 25)
    salt = "".join([random.choice("0123456789abcdef") for i in range(6)])
    return (n, salt)

def generate_ciphertext(random):
    n, salt = get_problem(random)
    trans = maketrans(alphabet, alphabet[n:] + alphabet[:n])
    ciphertext = ("easyctf{%s_%s}" % (flag, salt)).translate(trans)
    return StringIO(ciphertext)

def generate(random):
    return dict(files={
        "ciphertext.txt": generate_ciphertext()
    })

def grade(random, key):
    n, salt = get_problem(random)
    if key.find("%s_%s" % (flag, salt)) >= 0:
        return True, "Correct!"
    return False, "Nope."
```

In this way, not only is the shift randomly generated, so is the actual flag itself. Note that I use a helper function, `get_random` to actually generate the numbers to ensure that the numbers generated are the same for both the generation and the grading.

## Programming

For programming challenges, you'll need a separate `grader.py` and `generate.py` that behave a bit differently.

`generator.py` is responsible for generating each test case. It reads a single input: the test case number from stdin (you can use `int(input())` for that). The output (from stdout) will be passed to the user program as input.

`grader.py` acts like a "correct solution". It takes the same input that the user does (the output of generator) and produces the "correct answer" for that test case. The output of this program will be compared to the output of the user's program to determine correctness.

Overall the flow looks generally like

```ml
test case # ->
    -> generator.py ->
        -> (user's program) -> (user's answer) -+
        -> grader.py -> (correct answer) -------+
                                                v
                                             compare!
```

## Net Problems

If you need to serve a server program (**RAW TCP NOT WEB**), then include a `net` section in your `problem.yml` like this:

```yml
title: "Intro: TCP Server"
author: michael
category: Intro
value: 30
autogen: true
net:
  server: server.py
  port: 12481
```

This will serve using `server.py` (make sure it has the hash bang at the top) on port 12481, using xinetd. Read user input from stdin and send data to the user via stdout, xinetd will redirect this over sockets.

See the designated xinetd import tool for more details.

## Web Problems

more info incoming...