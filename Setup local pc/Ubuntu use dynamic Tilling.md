# Ubuntu Dynamic Tiling Setup

## Overview

This workstation uses **dynamic tiling** on Ubuntu through the **Pop!_OS Shell** GNOME extension.

Repository:
- https://github.com/pop-os/shell

The extension provides a tiling window manager experience while keeping the standard GNOME desktop. Windows are automatically arranged, while still allowing floating windows when needed.

---

## Why Pop!_OS Shell?

It provides many of the advantages of a traditional tiling window manager without replacing the desktop environment.

---

## Install

Follow installation steps in https://github.com/pop-os/shell#installation

---

## What to expect

This will change the windows behavior.
- Add a window tile configuration menu accessed by a tile symbol in the right top corner of the screen.
- Update GNOME key-binds assigned keys and add new key-binds.

---

## Keybindings

Most keybindings will be automatically set correctly after the installation, but some customization may be required to obtain the expected behavior:
- Some existing GNOME shortcuts may have to be updated
-  Some Pop!_OS Shell shortcuts may have to be updated
- Some **new custom keybindings** may have to be added in order to better fit the expected behavior.

---

## Custom Keyboard Shortcuts
In order to access custom keyboard shortcuts, access the following Ubuntu GNOME settings path:
**Settings → Keyboard → View and Customize Shortcuts**
The customize shortcuts scripts used are:
```
[custom-keybindings/custom0]
binding='<Super>Tab'
command='<path>/switch-to-next-workspace.sh'
name='Change workspace'

[custom-keybindings/custom1]
binding='<Shift><Super>Tab'
command='<path>/move-window-next-workspace.sh'
name='Change window from workspace'
```
See dynamic tilling custom keyboard shortcuts scripts source [code](https://github.com/aguidangla08/basic-scripts/tree/main/dynamic_tile_macros)

---

## Workspaces
Workspaces number may require to be updated in order to obtain the expected behavior, access the following Ubuntu GNOME settings path:
**Settings → Multitasking → Workspaces**

---

## Maintenance
Maintenance is simple for this feature:
- Review keyboard shortcuts after major Ubuntu or GNOME upgrades.
- Update this document whenever custom shortcuts or macro scripts change.