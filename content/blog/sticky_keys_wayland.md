+++
date = 2025-08-17
title = "Using Sticky Keys on Wayland without 'latchlock' Behavior"
description = "Using Sticky Keys on Wayland without 'latchlock' Behavior"
+++

### Introduction

I use sticky keys due to accessibility issues. One problem with the "sticky keys"
accessibility setting on Ubuntu is the default "latchlock" behavior. As explained in the `xkbset`
man page:

> a modifier pressed twice will be locked

So if you accidentally press a modifier key **twice** while typing, e.g. the control key, it will
lock the key leading to very frustrating and confusing behavior. The problem is explained in
detail in this [Ask Ubuntu question](https://askubuntu.com/questions/880740/disable-sticky-keys-locked-after-being-successively-pressed-twice-behavior).
This makes "sticky keys" virtually unusable for me on Ubuntu. This is true as of the latest
25.04 Ubuntu version. 

There is a confusing, outdated, and underdocumented amount of information on the internet about
this issue. Here I try to give some background and explain a working solution.

### Problem with XKB Set

If you are using X Window (X11) still, you can simply use XKB set to remove the default latchlock option.
Like so:

> xkbset sticky -twokey -latchlock

However, this does not work on Wayland. Ubuntu is moving to Wayland by default and in the future may
not come with X Window included. So this solution may not work in the future. Plus I like all the shinny
new features Wayland provides for the modern Ubuntu experience.

### Using XKB Directly

According to the Arch Linux Wiki [^1]: "The X keyboard extension, or XKB, defines the way keyboards codes are handled
in X..." Even though XKB is related to X, it also works on Wayland (the exact details of this are beyond the scope
of this post).

We will use XKB to set create our own sticky keys configuration. First, there are rules and symbols files under
`/usr/share/X11/xkb/rules/evdev` and `/usr/share/X11/xkb/symbols/`, respectively. As you might guess from the path,
these are used by xkb for defining keyboard behavior. Note: even though the paths reference X11, this still works with
Wayland. You can use the files in these directories as references or at least for keywords to look up.

We will be creating our own symbols and rules files in our home directory; this is cleaner than messing with the global
configs under `/usr`.

This solution may seem long, but it boils down to putting two files in the correct directory structure and
making the changes system-wide. **This solution works as of Ubuntu 25.04 running with Wayland**.

#### Step 1: Create Directory Structure

Create the following directory structure in your home directory (if it does not already exist):
```
> tree .config/xkb/
.config/xkb/
├── rules
│   └── evdev
└── symbols
    └── custom
```
Note: `rules/` and `symbols/` are directories. `evdev` and `custom` are files which we will create in the next step.

#### Step 2: Create `custom` file

Create a file called `custom` inside of `.config/xkb/symbols/`. Below are the contents using the `more` command. You
can find more info on how this file works by following this footnote[^2].
```
> more .config/xkb/symbols/custom
partial alphanumeric_keys
xkb_symbols "sticky" {
    key <RALT> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(mods=Alt,clearLocks) ]
    };
    key <LALT> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(mods=Alt,clearLocks) ]
    };
    key <LFSH> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(mods=Shift,clearLocks) ]
    };
    key <RTSH> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(mods=Shift,clearLocks) ]
    };
    key <RWIN> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(Mods=Super,clearLocks) ]
    };
    key <LWIN> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(Mods=Super,clearLocks) ]
    };
    key <LCTL> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(Mods=Control,clearLocks) ]
    };
    key <RCTL> {
        symbols = [ ISO_Level3_Shift ],
        actions = [ LatchMods(Mods=Control,clearLocks) ]
    };
};
```

Note: The name `custom` is arbitrary, but we will reference this name in another file. The name "sticky" is also
arbitrary.

#### Step 3: Create `evdev` file.

Create file `evdev` under `.config/xkb/rules/`. Here are the contents of mine using the `more` command:
```
> more .config/xkb/rules/evdev
! option    =   symbols
  custom:sticky     =   +custom(sticky)
  
! include %S/evdev
```
Here is where we reference our `custom` file name and our "sticky" name (if you use different names, switch them here).
The last line of our `evdev` file `! include %S/evdev` just includes the system-level `evdev` file. So we end up with
our rules plus the system rules.

#### Step 4:
The current way to set these changes in Ubuntu is using `gsettings`. This also makes it persistent across reboots and
shutdowns. From a terminal, run command:
```
gsettings set org.gnome.desktop.input-sources xkb-options '["custom:sticky"]'
```
Note: change the names on the last part of this command if you used different names.

### Troubleshooting
Originally the process above did not work for me even though I had worked on previous versions of Ubuntu. It turned
out that I had weird whitespace issues in the `evdev` file. I was using tabs instead of spaces. This caused Wayland
to fail on login, so I was stuck on a "log in loop". On restart Ubuntu was smart enough to switch from Wayland to X
without my knowledge.

Some troubleshooting tips:
- Use `echo $XDG_SESSION_TYPE` to ensure you are running wayland still.
- Warning: If you mess up the files above, Wayland may fail to start on login. Switch to X if that is an option, or
delete the `.config/xkb/` files as a last resort.


### Footnotes
[^1]: Arch Linux Wiki: [X keyboard extension](https://wiki.archlinux.org/title/X_keyboard_extension)
[^2]: Check out [this issue on the Sway Github repository](https://github.com/swaywm/sway/issues/6852#issuecomment-1352547670)
for more info on the `custom` file. This is how I got sticky keys working for my set-up (No Sway necessary!).

(_No AI was used in the making of this post_)