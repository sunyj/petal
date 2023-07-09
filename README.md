# Layered Python Environments

Petal is a single-script zero-dependency Python environment manager managing
[PEP 405](https://peps.python.org/pep-0405/) virtual environments as layers.

The name comes from an anagram of letters `L`, `A` (from layered), `P` (from
Python), `E`, `T` (from environment).

## Fast Q&A

### What exactly is a "Python environment"?

In this project, an "environment" means a [PEP
405](https://peps.python.org/pep-0405/) virtual environment as well as all
packages installed in it.

In most cases, "virtual" is omitted as it's the only type of Python
environment.  In some cases, we use **env** for short.

### Is petal a replacement to pip?

No.  In fact, all actual package operations, such as `install` and
`uninstall`, are delegated to pip.

### Then what's the point of a pip wrapper?

Petal uses pip, not just wraps it up.  Its distinctive features include:

- It is designed to be an environment-as-code (EaC) solution.
- It manages environments as layers.
- It supports environment delivery.
- It's zero-dependency and zero-config.  It does not even requires an
  installed pip.

### `pip freeze` with `requirements.txt` is already an EaC solution, isn't it?

It is, but not quite so. `requirements.txt` is only a **flat list** of
installed packages, without any dependency information.  Petal not only knows
which packages you ***need***, it also knows, and in fact focuses on, which
packages you ***want***.

# The Problems

From the experience of open-source communities in the past decades, to make a
community-driven language ecosystem thrive, we need at least these
foundational elements: a packaging standard, a package distribution hub, and a
dependency management tool.

Python seems to have everything. We can make an isolated virtual environment
with [PEP 405](https://peps.python.org/pep-0405/), then install packages to it
with [pip](https://pypi.org/project/pip/), which queries and downloads
packages from [PyPI](https://pypi.org/). For demo projects, that's
perfect. However, in real-life development and production, we sometimes feel
that pip isn't sophisticated enough to handle all the subtleties. Let's
discuss some (in my opinion) craved features in more detail.

## Regulated package removal

While pip installs all dependents when you install new packages, it does NOT
consult dependencies when you remove them. If you tried some package in your
project and finally decided to give it up, it could be a pain to clear up its
dependencies. Besides, the system may slip into dependency breaks easily.

## Environment as code

Package dependency is usually an indivisible part of a Python project, and it
would be nice to save the dependencies to a text file from which a virtual
environment can be constructed or restored.  That's the idea of the so-called
Environment-as-Code (EaC).

`pip freeze > requirements.txt` and `pip install -r requirements.txt` is
admittedly an EaC solution, but it's far from ideal.  Dependency structures
get lost in the roundtrip, so there's no way to tell which packages are the
ones you want, and which are their dependencies you have to install.

## Environment reuse

Suppose you're working on a application that requires some large packages
(such as `tensorflow`), and you need some other packages (such as `PyTest`) to
make unit tests.  It would be nice if the testing environment can reuse the
packages in development environment.

# A Solution

Judged by UNIX's golden "do one thing and do it well" philosophy, both pip and
PEP 405 are great tools with great implementation.  PEP 405 can make a
perfectly isolated virtual environment, to which
[pip installs packages](https://ianbicking.org/blog/2008/10/pyinstall-is-dead-long-live-pip.html).
An environment without packages is useless; Packages beyond an isolated
environment are clueless.  How to make them work together is a long quest for
the community.

Inspired by another UNIX philosophy: "make programs work together," we don't
need a replacement for pip; We need a tool that makes good use of this
important asset of the Python community and provides developers with a new
level of abstraction: environment management.

Petal is ambitious; It aims to be the last missing piece to complete the
puzzle of Python environment management.

# The Design

Let's put aside implementation details in this section and focus on petal's
semantic design.  New terms such as "base env", "core env", and "layer" will
be introduced with the unfolding of design ideas.

## Env Creation

Special thanks to PEP 405's
[elegant design](https://peps.python.org/pep-0405/#specification).

```bash
# make a new env with default name "env" in current directory
petal make

# of course you can name it
petal make myenv

# you can also choose python binary (quiz: which python is used by default?)
petal make myenv with /path/to/my/python

# myenv/.gitignore is generated by petal
git add myenv
```

## Layered Env

Petal can make a new env "on top" of other envs.  These envs are called the
***base envs***, or bases in short.

```bash
# usually only one base env
petal make test-env on dev-env

# you can also assign multiple base envs
petal make env on ../foo/env ../bar/env
```

At the very bottom lays petal's "core env", which is usually located in
`~/.cache/petal.py/core` and is bootstrapped automatically.  Core env contains
only `pip`.  All petal envs are based on the core env implicitily.  However,
package `pip` is hidden from all envs unless environment variable
`PETAL_USE_CORE` is set to a non-empty value.

```bash
petal make env
./env/bin/python -m pip  # Error: No module named pip
PETAL_USE_CORE=1 ./env/bin/python -m pip
```

A petal env is standard PEP 405 virtual env with two crucial hacks:

- A [path configuration (`.pth`) file](https://docs.python.org/3/library/site.html#)
  hack to tap into base envs (and their base envs recursively) for module searching.
- Two metadata files (`petal.json` and `petal.freeze`) to store the env's:
  - Python version (`major.minor`).
  - Base env(s).
  - Installed packages and their structures.

So, a petal env is also a ***layer***, which contains module paths of base
layers recursively.  Layered envs is the core mechanism of petal.  It enables
petal to manage an env's packages less intrusively.

## Env Recovery

The value of EaC lies in the ability to save env's changes into files and
manage them with version control.  petal command `make` is reused for env
recovery.

```bash
git clone .../my-proj
cd my-proj
petal make env
```

## Env Change and Env Structure

Packages can be installed, removed, or upgraded within a petal env with `petal
add`, `petal del`, and `petal upgrade` commands.

A package is ***native*** to an env only if the package is actually installed
in that env.  Packages installed in base layers are called ***included***
packages.

Packages installed explicitly through command line are called ***explicit
requirements***, or explicits for short.  Explicits are what user ***want***,
while packages they depend on are what user ***need***.  After every change,
explicits are recorded in `petal.json`, and all packages with versions are
written to `petal.freeze` under env's directory.

Strictly speaking, the semantics of `petal add` is not "install these packages
to env", but "add these packages to env, and install all dependencies if
needed".  Similar case for `petal del`.

The difference may sounds subtle, let's explain it with an example.

### The `pandas` and  `numpy` case

We all know that `pandas` depends on `numpy`, so what's the difference between
`petal add pandas` and `petal add pandas numpy`?  They all install the same
set of packages into the env, but they form different env ***structures***.

- `petal add pandas` adds only `pandas` into the explicits, so when you later
  run `petal del pandas`, all packages will be removed, as the explicits is an
  empty set after the removal.
- `petal add pandas numpy` adds both `pandas` and `numpy` into the explicts,
  so when you later run `petal del pandas`, all packages but `numpy` will be
  removed, as there's still `numpy` left in the explicit requirements set.

OK, then what happens if we add `numpy` after adding `pandas`?  Since `numpy`
has been installed on adding `pandas`, petal will simply skip the installation
and ***promote*** it into the explicit set.  What about the opposite?  What
happens when we remove `numpy` after that?  As it's still depended by
`pandas`, petal will not uninstall but ***delist*** it from explicits instead.

### Sophisticated Package Management

The above example reveals some sophistication of petal's package management
under the bland `add` and `del` commands.

| Env Change                    | Predicate     | Command           |
| ----------------------------- | ------------- | ----------------- |
| Install `<pkg>` to env        | add / install | `petal add <pkg>` |
| Remove `<pkg>` from env       | del / remove  | `petal del <pkg>` |
| Add `<pkg>` to explicits      | promote       | `petal add <pkg>` |
| Remove `<pkg>` from explicits | delist        | `petal del <pkg>` |

Petal env's explicit requirements and their dependencies form the structure of
the env.  Separating explicitly required packages from others is not only
necessary but also crucial, as it's the only way to enable petal to help users
focus on what they really ***want***, instead of what they ***need***.

### Package File Inclusion

If the `<pkg>` in command `petal add <pkg>` is a file path, either relative or
absolute, then the content of that file is used as package specs.  Each line
in the file should either be a package spec, or another file inclusion.  Even
if the file path is absolute, it is computed and saved as the relative path to
env's directory.

## Env Inspection

`petal show [env]` prints the aforementioned structure of the env.  By
default, only explicit and native packages are listed.  `petal show` command
supports these options:

- `-a`, `--all`: show all packages in env.
- `-d`, `--deep`: show packages in all base envs.
- `-v`, `--version`: show package version.

## Env Delivery

