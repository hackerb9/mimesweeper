# mimesweeper

Remove all current Wine MIME-types and file extension associations.
Attempt to prevent Wine from adding any more in the future.

Sweep away those annoying MIMEs before they explode!

## Usage

    mimesweeper [-vndk]

    -v: verbose
    -n: not-really (dry run)
    -d: daemonize
    -k: kill daemon

In general you will run **mimesweeper** with no arguments. It removes
files such as ~/.local/ share/mime/packages/x-wine*. It modifies the
user's local system.reg file so new associations will not be created.
(As long as the same WINEPREFIX directory is used, which is Wine's
default.)

If **mimesweeper** is run under **sudo**, it will also prevent MIME
associations even if you later create a new WINEPREFIX directory. It
does this by editing the system wine.inf files. However, due to Wine
keeping configuration files under /usr instead of /etc, this script
must be run again if wine is upgraded.

## Discussion

PROBLEMS, POSSIBLE SOLUTIONS, AND PITFALLS

### THE PROBLEM

By default, Wine hijacks
[MIME-types](https://en.wikipedia.org/wiki/MIME) and file
associations. That means when you click on a JPEG in your in your
filebrowser, it may fire up Internet Explorer to view it! This is not
only silly, but unnecessarily indiscriminate. I only want Wine
running when I explicitly ask to run a Microsoft Windows program,
never because there is some MIME-type it thinks it can handle.

In a perfect world, Wine wouldn't add associations by default. In an
_almost_ perfect world, Wine would have a system wide file —
say, /etc/wine/wine.inf — for local preferences. We live in neither and
must seek some other answer.

### First Solution
There is a checkbox option in **winecfg** to disable "Manage File
Associations".

#### First Solution's Problem

It has to be unchecked every time a new WINEPREFIX is created. This may work for other people, but happens way too often for me.

### Second Solution (Part A)

The Wine project has a configuration file — /usr/share/wine/wine.inf —
which allows one to set a systemwide preference by editing the
[Services] section and removing the "-a" switch from
winemenubuilder.exe. For example, one could edit it like so:

    sed -i~ 's/winemenubuilder.exe -a/winemenubuilder.exe/g'  /usr/share/wine/wine.inf

#### Second Solution (Part A)'s Problem

Unfortunately, Unix distributions such as Debian and Arch do not allow
configuration files under /usr, so every time a new version of Wine is
released, the wine.inf file gets clobbered with no option to merge
locally made changes.

### Second Solution (Part B)

Make changes as in (Part A), but use system package tools to prevent the
file from being clobbered. For Debian-based systems:

    dpkg-divert  --local  --rename  /usr/share/wine/wine.inf 
    cd /usr/share/wine
    cp  wine.inf.distrib  wine.inf 
    sed -i~ 's/winemenubuilder.exe -a/winemenubuilder.exe/g'  wine.inf

#### Second Solution (Part B)'s Problem

Important additions and changes to wine.inf from Wine developers will
not be merged during an upgrade, possibly causing breakage and
confusion.

### Second Solution Addendum: system.reg

Note that if wine had already been run in a WINEPREFIX before the
wine.inf is modified, then the user will have to also edit the
`$WINEPREFIX/system.reg` file in the same way.

### Third Solution (general)

Disable winemenubuilder.exe completely. (Three methods).

#### Third Solution (general)'s Problem

This also disables adding installed programs to the GUI menu. While an
acceptable tradeoff, it would be preferable to only disable the MIME
type associations.

### Third Solution (Part A)

Set an environment variable before running any installer to disable
winemenubuilder.exe:

  WINEDLLOVERRIDES=winemenubuilder.exe=d wine setup.exe

#### Third Solution (Part A)'s Problem

Requires user to remember the incantation for every single program
installer. I want to "set and forget" this.


### Third Solution (Part B)

Create a registry file to disable winemenubuilder.exe:

```bash
regedit <<< $'[HKEY_CURRENT_USER\Software\Wine\DllOverrides]\n"winemenubuilder.exe"=""\n'
```

#### Third Solution (Part B)'s Problem

Must be run for every new WINEPREFIX (which is every time for me).

### Third Solution (Part C)

Use UNIX host system's package tools to rename winemenubuilder.exe so
it cannot be run and will not be reinstalled at upgrade. For systems
that use the Debian package tools (dpkg, apt):

    dpkg-divert --local --rename /usr/lib/wine/winemenubuilder.exe.so
    dpkg-divert --local --rename /usr/lib64/wine/winemenubuilder.exe.so

#### Third Solution (Part C)'s Problem

Disables winemenubuilder completely, meaning applications are no
longer added to the desktop menus. Also, the instructions for doing
this depend on what flavor of UNIX is being used.

## Another way

All standard solutions are broken. Nevertheless, we can cobble a
Frankensteinian beast together from the previous attempts and make
something that is ugly but works. We need something that can:

1. Remove any existing Wine MIME associations.

1. Edit the wine.inf file (as in Second Solution) so that it always
   has the latest updates from the Wine developers, but is easy to
   fix when Wine is upgraded.

1. Handle new updates. Ideally, this would have been integrated with
   installation or upgrade of Wine, but that is not possible in the
   general case.

   Instead, we need a program that is easy to run from the command
   line, but can also be run from cron (no output by default, except
   errors). Moreover, it would be nice if it has the option to keep
   running in the background as a daemon and fix configuration files
   as they are changed during upgrades.

## Bugs

* Maybe I should have modified the Wine [source
  code](https://wiki.winehq.org/Source_Code) to read
  `/etc/wine/wine.inf` to allow local admin modifications. It'd be a
  cleaner solution.

* I don't personally use the -d (daemon) mode, so that code path is
  not as well tested.

* Daemonization makes a straight-forward script seem overly complicated.

* Waiting in the background for existing files to change may not even
  make sense if I'm creating new WINEPREFIXes and immediately
  installing software. I should instead use `inotifywait` to wait on
  the directory which contains all my WINEPREFIXes
  (`~/.wine/prefixes/`).

* The daemon edits files as soon as a change is detected. This may be
  incorrect if there is another process that is altering the file
  incrementally. (Race condition).

* `inotifywait` has only two modes: single-event or monitor.
  Single-event is quite easy to use from a shell-script, but always
  exits after the first (of possible several) events. Monitor mode is
  not worth using because it never quits and thus must be run as a
  coprocess. It would be nice if `inotifywait` had a "batch" mode
  which would return a group of events (instead of exiting on the
  first one). It could quit one second after the last event was
  received.
  