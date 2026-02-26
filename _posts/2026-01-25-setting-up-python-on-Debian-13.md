---
layout: post
title: "On Getting Up and Running with Pip in Debian Trixie"
description: "Setting up Python/Pip on a fresh Debian Trixie installation"
tags:
  - Blogging
---

## On Getting Up and Running with Pip in Debian Trixie

I finally gave in and restarted my blog from years ago, mainly due to the difficulty getting up and running with pip on the latest iteration of Debian Trixie (or Debian 13, for the numerical purists out there). Trixies originally released on 2025/08/13 and the image I'm using is from the most recent release (13.3, released 2026/01/10). This is, as of this time of writing, a capable release that's simply not kind to new Deb users who are accustomed to simply `sudo add-apt-repository` to add the `universe` repo and get running, you're going to have to get familiar with how `apt` does repository lists. 

---

#### The Background

On the upside, it's not as hard as you might think, but this is one of those edge cases where it's new enough that basically all of the answers on StackOverflow (and hence Copilot/ChatGPT, since those posts are the large bulk of the technical training corpus for those LLMs) will still persistently recommend that you run one of these commands despite them not working (unless you selected the full lists in the Trixie initialization process):

```powershell

# You have Python3, so update/upgrade and install pip with your flags to skip the Y/n.
sudo apt update -y && sudo apt upgrade -y
sudo apt install python3-pip
```

Unfortunately, you get this error: `Error: Unable to locate package Python3-pip`, since that's not in the default repository lists in Trixie. So you go to add the macro repo via `sudo add-apt-repository universe` repo like you would in Deb 12, right? No luck, because `add-apt-repository` is part of `software-properties-common`, which isn't present (yet) in Trixie because of [this bug](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1038747). This is one of the big downsides of open-source development when trying to hit a release date: there was a developer decision to deprioritize `software-properties-common` in favor of support for the new deb822 package sources list for APT. This is a bit ironic, since [deb822](https://repolib.readthedocs.io/en/latest/deb822-format.html) is the new format that will eventually deprecate the old repository lists format at `/etc/apt/sources.list`, while users will still have to go work in the repository lists because `apt-add-universe` was the util that provided the abstraction layer for adding new sources to APT locally. This is an even bigger issue if you're trying to include from a source that requires you import the GPG keys, but that's for another post.

#### The Fix

The fix that you *actually* want (presuming your user account is already a sudoer, because please don't just do all of your setup from root) is to add a set of default list repos to your list file. Open that with `sudo nano /etc/apt/sources.list`, add [these default lists](https://wiki.debian.org/SourcesList#Examples), save, and exit Nano:

```powershell
deb http://deb.debian.org/debian/ trixie main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian/ trixie main non-free-firmware contrib non-free

deb http://security.debian.org/debian-security trixie-security main non-free-firmware contrib non-free
deb-src http://security.debian.org/debian-security trixie-security main non-free-firmware contrib non-free

# bookworm-updates, to get updates before a point release is made;
# see https://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_updates_and_backports
deb http://deb.debian.org/debian/ trixie-updates main non-free-firmware contrib non-free
deb-src http://deb.debian.org/debian/ trixie-updates main non-free-firmware contrib non-free
```

Of note, both the Bookworm and Trixie examples will work in Trixie 13.3, just be sure to put them in the correct file based on format; debian.sources uses key-value pairs rather than the simple keyword text table format that sources.list uses.

Next, run your update/upgrade sequence and install python3-pip:

``` powershell
# Get all of the newly-included packages from the repository lists you added:
sudo apt update -y

# Check what can be upgraded, to make sure that you WANT to upgrade everything on the list:
apt list --upgradable

# Upgrade before installing your new packages:
sudo apt upgrade -y

# Install Python3-pip
sudo apt install python3-pip
```
There you go, now you have pip added to your primary Python installation on this host/machine, and you're off to the races!

---

#### A Word of Caution:

Now for a word of caution for the less experienced python developers: you have pip installed, and you can absolutely go wild with installing packages to your core Python installation, and running packages off that, but that's _really_ bad practice. If you want to upgrade a module/library to get to new functionality, you may also break some of your scripts that are running in production because they're using syntax/features that are deprecated in the newer iterations of those modules (if you're using Python, you probably know at least one or two of the many people who broke their Pandas 1.x and 2.x scripts when they upgraded to Pandas 3.0 without dependency version controls recently). This is why it's advised to set up a virtual environment for each project you work on, so that you can install your modules to exclusively that environment's Python installation, and you can use different versions of the package on different environments.

There are various Python-native tools you can use for this: `venv`, `virtualenv`, and `pyenv` are all tools for this purpose. I generally prefer and recommend `venv`.

---

