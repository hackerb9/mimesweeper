# mimesweeper

Remove all Wine file associations and attempt to prevent Wine from
adding any more in the future.

## Usage

    mimesweeper [-vndk]

    -v: verbose
    -n: not-really (dry run)
    -d: daemonize
    -k: kill daemon

## Discussion

PROBLEMS, POSSIBLE SOLUTIONS, AND PITFALLS

### THE PROBLEM

By default, Wine hijacks mimetypes and file associations. That means
when you click on a JPEG in your in your filebrowser, it may fire up
Internet Explorer to view it! This is not only silly, but feels like
an unnecessary risk: I only want Wine running when I explicitly ask to
run a Microsoft Windows program, never because there is some
[MIME-type](https://en.wikipedia.org/wiki/MIME) it thinks it can
handle.

In a perfect world, Wine wouldn't do this in the first place. In an
_almost_ perfect world, Wine would have a system wide file —
/etc/wine/wine.inf — for local preferences. We live in neither and
must find some other answer.

### First Solution
There is a checkbox option in `winecfg` to disable "Manage File
Associations".

### First Solution's Problem
It has to be unchecked every time a new WINEPREFIX is created. (Often, for me).

### Second Solution (A)

The Wine project has a configuration file — /usr/share/wine/wine.inf —
which allows one to set a systemwide preference by editing the
[Services] section and removing the "-a" switch from
winemenubuilder.exe. For example, one could edit it like so:

    sed -i~ 's/winemenubuilder.exe -a/winemenubuilder.exe/g'  /usr/share/wine/wine.inf

### Second Solution (A)'s Problem

Unfortunately, Unix distributions such as Debian and Arch do not allow
configuration files under /usr, so every time a new version of Wine is
released, the wine.inf file gets clobbered with no option to merge
locally made changes.

### Second Solution (B)

Make changes as in (A), but use system package tools to prevent the
file from being clobbered. For Debian-based systems:

  dpkg-divert  --local  --rename  /usr/share/wine/wine.inf 
  cd /usr/share/wine
  cp  wine.inf.distrib  wine.inf 
  sed -i~ 's/winemenubuilder.exe -a/winemenubuilder.exe/g'  wine.inf

### Second Solution (B)'s Problem

Important additions and changes to wine.inf from Wine developers will
not be merged during an upgrade, possibly causing breakage and
confusion.

### Second Solution Addendum: system.reg

Note that if wine had already been run in a WINEPREFIX before the
wine.inf is modified, then the user will have to also edit the
`$WINEPREFIX/system.reg` file in the same way.

### Third Solution (general)

Disable winemenubuilder.exe completely. (Three methods).

### Third Solution (general)'s Problem

This also disables adding installed programs to the GUI menu. While an
acceptable tradeoff, it would be preferable to only disable the MIME
type associations.

### Third Solution (A)

Set an environment variable before running any installer to disable
winemenubuilder.exe:

  WINEDLLOVERRIDES=winemenubuilder.exe=d wine setup.exe

### Third Solution (A)'s Problem

Requires user to remember the incantation for every single program
installer. I want to "set and forget" this.


### Third Solution (B)

Create a registry file to disable winemenubuilder.exe:

  regedit <<< $'[HKEY_CURRENT_USER\Software\Wine\DllOverrides]\n"winemenubuilder.exe"=""\n'

### Third Solution (B)'s Problem

Must be run for every new WINEPREFIX (which is every time for me).

### Third Solution (C)

Use UNIX host system's package tools to rename winemenubuilder.exe so
it cannot be found and will not be reinstalled at upgrade. For systems
that use the Debian package tools (dpkg, apt):

  dpkg-divert --local --rename /usr/lib/wine/winemenubuilder.exe.so
  dpkg-divert --local --rename /usr/lib64/wine/winemenubuilder.exe.so

### Third Solution (C)'s Problem

Disables winemenubuilder completely, meaning applications are no
longer added to the desktop menus. Also, the instructions for doing
this depend on what flavor of UNIX is being used.

### Fourth Solution

There is no fourth solution.

Nevertheless, we can cobble a Frankensteinian beast together from the
previous attempts and make something that is ugly but works.

A. Remove any existing Wine MIME associations.

B. Edit the wine.inf file (as in Second Solution) so that it always
   has the latest updates from the Wine developers, but is easy to
   fix when Wine is upgraded.

C. Handle new updates. Ideally, this would be integrated with
   installation or upgrade of Wine, but we can't do that easily.

   Instead, this script should be able to be run from cron (no
   output except errors). Moreover, it would be nice if it could
   keep running in the background and use inotifywait to check for
   changes.

