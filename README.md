# sedparse

- Author: Aurelio Jargas
- License: GPLv3
- Tested with Python 2.7, 3.4, 3.5, 3.6 and 3.7 (see [.travis.yml](https://github.com/aureliojargas/sedparse/blob/master/.travis.yml))

A translation from C to Python of GNU sed's parser for sed scripts.

After running sedparse in your sed script, the resulting "[AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)" is available in different formats:

- List of objects (translated C structs)
- List of dictionaries
- JSON

For a complete reference on how the different sed commands are mapped by the parser, see:

- [tests/reference.sed](https://github.com/aureliojargas/sedparse/blob/master/tests/reference.sed) - original sed script
- [tests/reference.json](https://github.com/aureliojargas/sedparse/blob/master/tests/reference.json) - JSON generated by sedparse


## About the translation

I copied the original code in C and translated everything to Python, line by line.

To make it feasible to keep this code updated with future GNU sed code, this is a literal translation, trying to mimic as much as possible of the original code. That includes using the same API, same logic, same variable and method names and same data structures. Pythonic code? Sorry, not here.

The accuracy of the parser is checked by extensive unit tests in [tests/](https://github.com/aureliojargas/sedparse/tree/master/tests).

Sedparse was translated from this GNU sed version:

http://git.savannah.gnu.org/cgit/sed.git/commit/?id=a9cb52bcf39f0ee307301ac73c11acb24372b9d8

    commit a9cb52bcf39f0ee307301ac73c11acb24372b9d8
    Author: Assaf Gordon <assafgordon@gmail.com>
    Date:   Sun Jun 2 01:14:00 2019 -0600

> Note that this is not a full GNU sed implementation.
> Only the parser for sed scripts was translated.
> Check https://github.com/GillesArcas/PythonSed for a working sed in Python.


## Sedparse extensions to the original parser

- Preserves comments
- Preserves blank lines between commands
- Preserves original flags for the `s` command
- Preserves original flags for regex addresses


## Installation

    pip install --user sedparse
    sedparse --help

Alternatively, you can just download and run the [sedparse.py](https://raw.githubusercontent.com/aureliojargas/sedparse/master/sedparse.py) file, since it is self-contained with no external dependencies.


## Usage from the command line

The informed sed script will be parsed and checked for syntax errors. If everything is fine, a JSON representation of the script is sent to STDOUT.

Just like in sed, you can inform the sed script using one or more `-e` options:

```console
$ python sedparse.py -e "s/foo/bar/g" -e "5d"
[
    {
        "cmd": "s",
        "line": 1,
        "x": {
            "cmd_subst": {
                "regx": {
                    "flags": "g",
                    "pattern": "foo",
                    "slash": "/"
                },
                "replacement": {
                    "text": "bar"
                }
            }
        }
    },
    {
        "a1": {
            "addr_number": 5,
            "addr_type": 3
        },
        "cmd": "d",
        "line": 1
    }
]
$
```

Or you can inform the sed script as a file argument using `-f`:

```console
$ echo '1,10!d' > head.sed
$ python sedparse.py -f head.sed
[
    {
        "a1": {
            "addr_number": 1,
            "addr_type": 3
        },
        "a2": {
            "addr_number": 10,
            "addr_type": 3
        },
        "addr_bang": true,
        "cmd": "d",
        "line": 1
    }
]
$ rm head.sed
$
```

Or even as text coming from STDIN when using the special `-` file:

```console
$ echo '\EXTREMITIES' | python sedparse.py -f -
[
    {
        "a1": {
            "addr_regex": {
                "flags": "MI",
                "pattern": "XTR",
                "slash": "E"
            },
            "addr_type": 2
        },
        "cmd": "T",
        "line": 1,
        "x": {
            "label_name": "IES"
        }
    }
]
$
```


## Usage as a Python module

Use `sedparse.compile_string()` to parse a string as a sed script. You must inform a list that will be appended in-place with the parsed commands.

```python
>>> import sedparse
>>> sedscript = """\
... 11,/foo/ {
...     $!N
...     s/\\n/-/gi
... }
... """
>>> parsed = []
>>> sedparse.compile_string(parsed, sedscript)
>>>
```

Each sed command is represented by a `struct_sed_cmd` instance.

```python
>>> import pprint
>>> pprint.pprint(parsed)  # doctest:+ELLIPSIS
[struct_sed_cmd(line=1, cmd='{', ...),
 struct_sed_cmd(line=2, cmd='N', ...),
 struct_sed_cmd(line=3, cmd='s', ...),
 struct_sed_cmd(line=4, cmd='}', ...)]
>>>
```

You can `str()` each command, or any of its inner structs, to get their "human readable" representation.

```python
>>> [str(x) for x in parsed]
['11,/foo/ {', '$ !N', 's/\\n/-/gi', '}']
>>> str(parsed[0])
'11,/foo/ {'
>>> str(parsed[0].a1)
'11'
>>> str(parsed[0].a2)
'/foo/'
>>>
```

Use `.to_dict()` to convert a command into a Python dictionary.

```python
>>> cmd_n = parsed[1]
>>> str(cmd_n)
'$ !N'
>>> pprint.pprint(cmd_n.to_dict())
{'a1': {'addr_type': 7}, 'addr_bang': True, 'cmd': 'N', 'line': 2}
>>>
>>> pprint.pprint(cmd_n.to_dict(remove_empty=False))
{'a1': {'addr_number': 0, 'addr_regex': None, 'addr_step': 0, 'addr_type': 7},
 'a2': None,
 'addr_bang': True,
 'cmd': 'N',
 'line': 2,
 'x': {'cmd_subst': {'outf': {'name': ''},
                     'regx': {'flags': '', 'pattern': '', 'slash': ''},
                     'replacement': {'text': ''}},
       'cmd_txt': {'text': []},
       'comment': '',
       'fname': '',
       'int_arg': -1,
       'label_name': ''}}
>>>
```

Use `.to_json()` to convert a command into JSON.

```python
>>> print(cmd_n.to_json())
{
    "a1": {
        "addr_type": 7
    },
    "addr_bang": true,
    "cmd": "N",
    "line": 2
}
>>>
```

Have fun!

```python
>>> [x.cmd for x in parsed]  # list of commands
['{', 'N', 's', '}']
>>> [str(x) for x in parsed if x.a1 is None]  # commands with no address
['s/\\n/-/gi', '}']
>>> [str(x) for x in parsed if x.addr_bang]  # commands with bang !
['$ !N']
>>> [x.x.comment for x in parsed if x.x.comment]  # extract all comments
[]
>>> [x.x.fname for x in parsed if x.cmd in "rRwW"]  # list of read/write filenames
[]
>>>
```
