# Layered Python Environments

`petal` is a single-script zero-dependency Python package manager managing
[PEP 405](https://peps.python.org/pep-0405/) virtual environments as layers.

The name comes from an anagram of letters `L`, `A` (from layered), `P` (from
Python), `E`, `T` (from environment).

## Fast Q&A

### What does "Python environment" really mean?

In this document, an "environment" means a [PEP
405](https://peps.python.org/pep-0405/) virtual environment as well as all
packages installed in it.  In some cases, we use **env** for short.

### Is `petal` a replacement to `pip`?

No.  In fact, all package actions, such as install, uninstall, upgrade, are
delegated to `pip`.

### Then what's the point of a `pip` wrapper?

`petal` uses `pip`, not just wraps it up.  Its distinctive features include:

- It is designed to be an environment-as-code (EaC) solution.
- It manages environments as layers.
- It supports environment delivery.
- It's zero-dependency and zero-config.  It does not even requires an
  installed `pip`.

### `pip freeze` with `requirements.txt` is already an EaC solution, isn't it?

It is, and it is not. `requirements.txt` is only a **flat list** of installed
packages, without any dependency information.  `petal` not only knows which
packages you **need**, it also knows, and in fact focuses on, which packages
you **want**.

# The Problems

From the experience of open-source communities in the past decades, to make a
community-driven language ecosystem thrive, we need at least these
foundational elements: a packaging standard, a package distribution hub, and a
dependency management tool.

Python seems to have everything. We can make an isolated virtual environment
with [PEP 405](https://peps.python.org/pep-0405/), then install packages to it
with [`pip`](https://pypi.org/project/pip/), which queries and downloads
packages from [PyPI](https://pypi.org/). For demo projects, that's
perfect. However, in real-life development and production, we sometimes feel
that `pip` isn't sophisticated enough to handle all the subtleties. Let's
discuss some (in my opinion) craved features in more detail.

## Regulated package removal

While `pip` installs all dependents when you install new packages, it does NOT
consider dependencies when you remove them.  If you tried a package in your
project and finally decided to give it up, it could be a pain to clear up its
dependencies.  Besides, the system may slip into dependency breaks easily.

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
[`pip` installs packages](https://ianbicking.org/blog/2008/10/pyinstall-is-dead-long-live-pip.html).
An environment without packages is useless; Packages beyond an isolated
environment are clueless.  How to make them work together is a long quest for
the community.

Inspired by another UNIX philosophy: "make programs work together," we don't
need a replacement for pip, but a tool that makes good use of this important
asset of the Python community and provides developers with a new level of
abstraction: environment management.

`petal` is ambitious.  It aims to be the last missing piece of the Python
environment puzzle.
