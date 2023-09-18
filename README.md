# Layered Python Environments

Petal is a single-script zero-dependency Python environment manager managing [PEP 405](https://peps.python.org/pep-0405/) virtual environments as layers.

The name comes from an anagram of letters `L`, `A` (from layered), `P` (from Python),  `E`, `T` (from environment).

## Quick Start

1. Download latest [petal](https://raw.githubusercontent.com/sunyj/petal/master/petal).  It's just a single Python script.
2. Run it without any command line arguments.  The help page it prints also serves as the complete manual.
3. Experiment with creating an environment and making changes to it.  Check the structure of your environment with `petal show`.

```
$ curl -o petal https://raw.githubusercontent.com/sunyj/petal/master/petal
$ chmod +x petal

$ ./petal make
download pip 23.2.1 from ... ...
pip bootstrap done for ~/.cache/petal.py/core/python3.8
petal core (python3.8) made: ~/.cache/petal.py/core/python3.8
environment env made on top of [~/.cache/petal.py/core/python3.8] with /usr/bin/python3.8

$ ./petal add pandas
... ...
$ ./petal show
packages in ~/env:
- pandas

$ ./petal add numpy
[~/env] promote numpy as explicit
$ ./petal show
packages in ~/env:
- pandas
- numpy

$ ./petal del pandas
$ ./petal show -a
```

## Fast Q&A

### What exactly is a "Python environment"?

In this project, an "environment" means a [PEP 405](https://peps.python.org/pep-0405/) virtual environment as well as all packages installed in it.

In most cases, "virtual" is omitted for concision.  In some contexts, we use **env** for short.

### Is petal a replacement to pip?

No, and probably never.

In fact, all actual package operations, such as `install` and `uninstall`, are delegated to pip.

### Then what's the point of a pip wrapper?

Petal is NOT a wrapper of pip, it leverages pip for pckage installation and dependency query.  Its distinctive features include:

- An environment-as-code (EaC) solution.
- Manage environments as layers.
- Automatic removal of unnecessary dependent packages.
- Environment delivery.
- Zero-dependency, zero-config.


### `pip freeze` with `requirements.txt` is already an EaC solution, isn't it?

It is, but not entirely. While `requirements.txt` provides a list of installed packages, it lacks the necessary dependency information.  It is a planar projection of a higher dimensional structure, which is insufficient for proper management of that structure.  In contrast, Petal tracks not only ***what*** packages were installed but also ***why*** you have them installed.

# The Problems

From the experience of open-source communities in the past decades, to make a community-driven language ecosystem thrive, we need at least these foundational elements: a packaging standard, a package distribution hub, and a dependency management tool.

Python seems to have everything. We can make an isolated virtual environment with [PEP 405](https://peps.python.org/pep-0405/), then install packages to it with [pip](https://pypi.org/project/pip/), which queries and downloads packages from [PyPI](https://pypi.org/). For demo projects, that's perfect. However, in real-life development and production, we sometimes feel that pip isn't sophisticated enough to handle all the subtleties. Let's discuss some (in my opinion) craved features in more detail.

## Regulated package removal

While pip installs all dependents when you install new packages, it does NOT consult dependencies when you remove them. If you tried some package in your project and finally decided to give it up, it could be a pain to clear up its dependencies. Besides, the system may easily slip into dependency breaks.

## Environment as code

Package dependency is usually an indivisible part of a Python project, and it would be nice to save the dependencies to a text file from which a virtual environment can be constructed or restored.  That's the idea of the so-called Environment-as-Code (EaC).

`pip freeze > requirements.txt` and `pip install -r requirements.txt` is admittedly an EaC solution, but it's far from ideal.  Dependency structures get lost in the roundtrip, so there's no way to tell which packages are the ones you ***want*** and which are their dependencies you ***need*** to install.

## Environment reuse

Suppose you're working on a application that requires some large packages (such as `tensorflow`), while you need some other packages (such as `PyTest`) to make unit tests.  It would be nice if the testing environment can reuse the packages in development environment.

# A Solution

Both PEP 405 and pip are great examples of UNIX's famous "do one thing and do it well" philosophy.  PEP 405 can make a perfectly isolated virtual environment, to which [pip installs packages](https://ianbicking.org/blog/2008/10/pyinstall-is-dead-long-live-pip.html).  Environments without packages are useless; Packages beyond an isolated environment are clueless.  How to make them work together has been a long quest for the community.

Inspired by another UNIX philosophy: "make programs work together," I gradually realized that we don't need a replacement for pip.  Instead, we need a tool that makes good use of these core assets of the Python community to implement a new level of abstraction: environment management.

Petal is ambitious; It aims to be the last missing piece to the puzzle of Python environment management.

# The Design

Let's put aside implementation details in this section and focus on petal's semantic design.  New terms such as "base env", "core env", and "layer" will be introduced with the unfolding of design ideas.

## Env Creation

Special thanks to PEP 405's [elegant design](https://peps.python.org/pep-0405/#specification).

```bash
# make a new env with default name "env" in current directory
petal make

# of course you can name it
petal make dev-env

# you can also choose python binary (quiz: which python is used by default?)
petal make dev-env with /path/to/my/python

# dev-env/.gitignore is generated by petal
git add dev-env
```

## Env Layers

Petal can make a new env "on top" of other envs.  These envs are called the ***base envs***, or bases in short.

```bash
# usually only one base env
petal make test-env on dev-env

# you can also assign multiple base envs
petal make env on ../foo/env ../bar/env
```

At the very bottom lays petal's "core env", which is usually located in `~/.cache/petal.py/core` and is bootstrapped automatically.  Core env contains only `pip`.  All petal envs are based on the core env implicitily.  However, the `pip` package is hidden from all envs unless environment variable `PETAL_USE_CORE` is set to a non-empty value.

```bash
petal make env
./env/bin/python -m pip  # Error: No module named pip
PETAL_USE_CORE=1 ./env/bin/python -m pip
```

A petal env is standard PEP 405 virtual env with two crucial hacks:

- A [path configuration (`.pth`) file](https://docs.python.org/3/library/site.html#) hack to tap into base envs (and their base envs recursively) for module searching.
- Two metadata files (`petal.json` and `petal.freeze`) to store the env's:
  - Python version (`major.minor`).
  - Base env(s).
  - Installed packages and their structures.

So, a petal env is also a ***layer***, which contains module paths of base layers recursively.  Layered envs is the core mechanism of petal.  It enables petal to manage an env's packages less intrusively.

## Env Recovery

The value of EaC lies in the ability to save env's changes into files and manage them with version control.  petal command `make` is reused for env recovery.

```bash
git clone ... ... ... /my-proj.git
cd my-proj
petal make env
```

## Env Change and Env Structure

Packages can be installed, removed, or upgraded within a petal env with `petal add`, `petal del`, and `petal upgrade` commands.

A package is ***native*** to an env only if the package is actually installed in that env.  Packages installed explicitly through command line are called ***explicit requirements***, or explicits for short.  After every change, all native packages are recorded to `<env>/petal.freeze` in `pip freeze` style, while all native explicits are recorded to `<env>/petal.json`.

Strictly speaking, the semantics of `petal add` is not exactly "install these packages to env", but "add these packages to env, and install their dependencies if needed".  Similar case for `petal del`.

The difference may sounds subtle, let's explain it with an example.

### The `pandas` and  `numpy` case

We all know that `pandas` depends on `numpy`, then what's the difference between `petal add pandas` and `petal add pandas numpy`?  They all install the same set of packages into the env, but they form different env ***structures***.

- `petal add pandas` adds only `pandas` into the explicits, so when you later run `petal del pandas`, all packages will be removed, as the explicits will be an empty set after the removal.
- `petal add pandas numpy` adds both `pandas` and  `numpy` into the explicts, so when you later run `petal del pandas`, all packages but `numpy` will be removed, as there's still `numpy` left in the explicit requirements set.

OK, then what happens if we add `numpy` after adding `pandas`?  Since `numpy` has been installed on adding `pandas`, petal will simply skip the installation and ***promote*** it into the explicit set.  What about the opposite?  What happens when we remove `numpy` after that?  As it's still depended by `pandas`, petal will not uninstall but ***delist*** it from explicits instead.

### Sophisticated Package Management

The above example reveals some sophistication of petal's package management under the bland `add` and `del` commands.

| Env Change                    | Predicate     | Command           |
| ----------------------------- | ------------- | ----------------- |
| Install `<pkg>` to env        | add / install | `petal add <pkg>` |
| Remove `<pkg>` from env       | del / remove  | `petal del <pkg>` |
| Add `<pkg>` to explicits      | promote       | `petal add <pkg>` |
| Remove `<pkg>` from explicits | delist        | `petal del <pkg>` |

Petal env's explicit requirements and their dependencies form the structure of the env.  Separating explicitly required packages from others is not an over design but a crucial yet simple mechanism for users to focus on what they really ***want***, instead of a mixture of everything.

Slogan candidate: "Focus on what you want, let petal handle the rest."

### Plan Before Change

You can prepend `plan` to `add`, `del`, and `upgrade` commands to check what petal plans for these change actions before you really perform them.

## Env Inspection

`petal show [env]` prints the aforementioned structure of the env.  By default, only explicit and native packages are listed.  `petal show` command supports these options:

- `-a`, `--all`: show all packages in env, not just explicits.
- `-v`, `--version`: show packages' versions.

## Env Delivery

Although petal claims to create and maintain standard PEP 405 virtual environments, as we mentioned earlier, petal has to make some hacks to the environment to support EaC and layered env.  Besides, in a multi-layer setup, packages may be installed in different layers.  All these facts are not the most friendly to application delivery.

In fact, virtual env based Python application delivery is an important driving factor behind petal and its predecessors.  While env delivery is usually not exactly application delivery, it is always a crucial part.  To that end, petal provides the `deliver` command to facilitate the delivery of applications developed in petal environments.

Env delivery is as simple as putting an elephant into a fridge, with only two steps:

1. Create a clean PEP 405 destination env without any petal hacks.
2. Copy all the packages from the source to the destination, including those in the base env(s), recursively.

# The Hacks

Petal is the fruit that grows out of a tree of experimental projects I occasionally fiddled with over the last two years (2022-23). I used them in almost all my Python projects and feel quite satisfied. I am grateful for Python, its powerful standard library, and its persistence in pursuit of elegant designs.  Petal's technical choices and implementation details are explained in the following sections.

## Layered Virtual Environment

Two files, `_petal_layers.py` and `petal-layers.pth`, are created in the site library path on creation of a new petal env.  It utilizes Python's [path configuration hook](https://docs.python.org/3/library/site.html#) to append entries to `sys.path` according to metadata in  `<env>/petal.json`, recursively.  Please refer to `install_layers_hack` method in the source code for more details.

## How is pip used in petal?

If Python's path configuration hook is the first enabling technology of petal, then pip as a zero-dependency and runnable module is the second. Under the layered env mechanism, petal can install pip module in the bottom-most, so-called "core env", which is shared by all petal envs using the same Python minor version.  The core env, or the "pip layer", is only visible to upper layers when the environment variable `PETAL_USE_CORE` is non-empty.

The initial installation of the pip module, which is conventionally called bootstrapping, is done by downloading its latest wheel distribution from PyPI, and installing it directly with Python's [`zipimport`](https://docs.python.org/3/library/zipimport.html) feature.  Two popular PyPI mirrors in mainland China area is added to the mirrors list along with the main site.  Trial connections are made in parallel (with [`asyncio`](https://docs.python.org/3/library/asyncio.html)) to all mirror sites before every bootstrapping, and the one with minimum TCP connection latency is chosen.

Petal uses pip in two ways:  `pip` command execution and package dependency inquiry.  Module `pip._internal.metadata.pkg_resources` is imported to list installed packages with their dependencies.  Read the source code of `PetalEnv` class for more details.

## Package management delegation

All actual package changing (add, del, upgrade) actions are performed by `pip` command, using the pip module in the core env.  All `pip install` command switches are supported.

## Command Aliases

Petal is sophisticated, not complicated, and I want to deliver that message through a lucid command line design.  A side effect of this design is the convenience to support command aliases.  To honor traditional command naming conventions, petal supports these command aliases.

| Alias             | Command         |
| ----------------- | --------------- |
| `petal install`   | `petal add`     |
| `petal uninstall` | `petal del`     |
| `petal remove`    | `petal del`     |
| `petal copy`      | `petal deliver` |

# Limitations

- Only tested on Linux with Python 3.8.
- Should work on MacOS, but no extensive tests yet.
- Unlikely to work properly on Windows for now, but should be fairly easy to make it work as petal only uses the standard library.
