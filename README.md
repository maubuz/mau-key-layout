# mau-keyboard-customization
Repositiory with my keyboard shortcuts and hotkeys for Linux and Windows

## Windows AutoHotKey (AHK)

This is Mau's AHK setup.

### Installing AHK

1. Go to official site at https://www.autohotkey.com/ , download and install the application
2. Download the script in this repository from windows-ahk/
3. Run the script from AHK.

## Linux XKB Key remapping
This document is made for Fedora 33 on Gnome

### xkb setting locations
The xkb configuration seems to be located in `/usr/share/X11/xkb`
![50171832efff32ad40b3cb2b9ab63f04.png](assets/50171832efff32ad40b3cb2b9ab63f04.png)

### Listing current xkb settings
The following commands output the currently used xkb settings:
`setxkbmap -query`
`setxkbmap -print`
`xprop -root | grep XKB`

In order to list all the rules available in evdev:
`localectl list-x11-keymap-options | cat`

### Creating a custom xkb setting

#### Method 1: Simple Caps for level3 (not consistent)

Initially, I created a custom setting by follow the proceduce in the forum [How can I make backspace act as escape using setxkbmap?](https://unix.stackexchange.com/questions/212573/how-can-i-make-backspace-act-as-escape-using-setxkbmap)

Steps:

1. Create a new file in `/usr/share/X11/xkb/symbols/` called `mau`
2. In this file create the desired symbols configuration. In the example below this a setting called "mau_keys" was created in the `mau` file:
```
partial alphanumeric_keys
xkb_symbols "mau_keys" {

// Arrow keys

    key <AD08> { [  i,  I,  Up] };

    key <AC07> { [  j,  J,  Left] };
    
    // ... Other keys
};
```
3. Create a new option in the file `xkb/rules/evdev`. Locate the section `! option = symbols` and add:
```
    ! option    =   symbols
      mau:mau_keys  =   +mau(mau_keys)
```

4. Create an option description in `xkb/rules/evdev.lst` under `! option` (e.g. right before `ctrl`):

```
mau                Custom keys for Mau
mau:mau_keys	   Shortcuts for arrows, select all, cut, copy, paste, closing windows and more 
```

5. Apply the newly created partial symbols file `mau`: 
```
setxkbmap -layout us -option mau:mau_keys
```
The `setxkbmap` command above will active the new configuration, however, it will not persist when the system is rebooted. To make the xkb_smbols file persistent, see section **Persistent xkb options in Gnome** below.

6. Enalble the pre-existing xkb option "Caps Lock chooses level three symbols ". This option is `lv3:caps_switch` (already located in ) `/usr/share/X11/xkb/symbols/` and can be turned on from _Gnome Tweaks > Keyboard and Mouse > Additional Layout Options > Key to choose the 3rd level > Caps Lock

##### Limitations of Method 1
These settings will work most of the time, however, it has two limitations:
1. Will not be consistent accross all applications. This problem is well described in a few different locations:
    - From the Arch Wiki, [X keyboard extension: Caps hjkl as vimlike arrow keys](https://wiki.archlinux.org/index.php/X_keyboard_extension#Caps_hjkl_as_vimlike_arrow_keys)

> Creating keymappings that clear modifiers from the keypress is necessary if the target key is to be used in keyboard shortcuts. For instance, highlighting text from the main keyboard (`Shift+Left`), or changing chats in most messengers (`Alt+Down`) will not work if there is an additional `Caps` modifier sent.

- From the the xkeyboard-config project repository, section _3.1 Levels And Groups_ of their [README.enhancing](https://gitlab.freedesktop.org/xkeyboard-config/xkeyboard-config/-/blob/master/docs/README.enhancing) documentation:
> Remember that it is the application which must decide which symbol matches which keycode according to effective modifier state. The X server itself sends only an input event message to. Of course, usually the general inter-pretation is processed by Xlib, Xaw, Motif, Qt, Gtk and similar libraries. The X server only supplies its mapping table (usually upon an application startup).

2. Clearing and adding Mod keys for more complex hot keys is not possible.
    - For example, _Caps + d_ cannot send _Ctrl + c_ (copy)
    - This is related to the previous point. In order to press _Caps + d_ and tranfer it as _Ctrl + c_ to the application in question, the modifier _Caps_ would have to be "cleared" and the modifier _Ctrl_ would have to be "inserted" to the keypress event. This seems to only be possible with method 2 below.

#### Method 2: Redirecting keys & Clearning Caps Modifier

Method two follows procedure described in the Arch Wiki, [X keyboard extension: Caps hjkl as vimlike arrow keys](https://wiki.archlinux.org/index.php/X_keyboard_extension#Caps_hjkl_as_vimlike_arrow_keys):

1. Find the file `xkb/types/complete` and add the following to the bottom:
```
xkb_types "complete" {
   ...
   type "CUST_CAPSLOCK" {
       modifiers= Shift+Lock; 
       map\[Shift\] = Level2;            //maps shift and no Lock. Shift+Alt goes here, too, because Alt isn't in modifiers.
       map\[Lock\] = Level3;
       map\[Shift+Lock\] = Level3;       //maps shift and Lock. Shift+Lock+Alt goes here, too.
       level_name\[Level1\]= "Base";
       level_name\[Level2\]= "Shift";
       level_name\[Level3\]= "Lock";
   };
 };
```

2. Change caps from a lock (toggle) to a set (press) by modifying the already existing definition in `xkb/compat/complete`from LockMods to SetMods:

```
xkb_compatibility "complete" {
   ...
   interpret Caps_Lock+AnyOfOrNone(all) {
       action= SetMods(modifiers=Lock);
   };
   ...
 };
```

3. Create a new symbols file `mau` in `/usr/share/X11/xkb/symbols/` with the desired keys and their multiple levels. This is the same as step 2 of Method 1 above.
    Note the following:
    - You can call this file whatever you want. I choose to call it `mau`
    - The complete file is available in this repository under the `linux-xkb` folder. Below is a snipped
    - The section `mau_keys` is one sub-component defined in the file `mau`. It's possible to declare multiple sub-components and load them individually.
```
partial alphanumeric_keys

xkb_symbols "mau_keys" {

    key.type = "CUST_CAPSLOCK";

// Arrow keys

    key <AD08> {
	symbols[Group1]= [               i,               I,               Up],
        actions[Group1]= [      NoAction(),      NoAction(),   RedirectKey(keycode=<UP>, clearmods=Lock) ]
        };

// ... Other arrow keys

// Function keys

    key <AB06> {       
    	symbols[Group1]= [               n,               N,               BackSpace],
	actions[Group1]= [      NoAction(),      NoAction(),   RedirectKey(keycode=<BKSP>, clearmods=Lock) ]
	};

// ... other function keys 

// Hotkey shortcuts

   key <AC01> {
        symbols[Group1]= [	  	a,		A,		a	],
        actions[Group1]= [      NoAction(),      NoAction(),   RedirectKey(keycode=<AC01>, clearmods=Lock, modifiers=Control) ]
        };

   key <AD03> {  // Caps + E simulates Alt + F4
        symbols[Group1]= [	  	e,		E,		F4	],
        actions[Group1]= [      NoAction(),      NoAction(),   RedirectKey(keycode=<FK04>, clearmods=Lock, modifiers=Alt) ]
        };
        
// ... other hotkeys        

};
```

The "magic" of this method happens through the _RedirectKey_ action. As described in Doug Palmer's _An Unreliable Guide to XKB Configuration_ in the [section Compatibility Maps](https://www.charvolant.org/doug/xkb/html/node5.html#SECTION00055000000000000000),  and in Ivan Pascal's notes on X Keyboard Extension, section [Another key press emulation](https://web.archive.org/web/20070206114753/http://pascal.tsu.ru/en/xkb/gram-action.html#key), the action _RedirectiKey_ can emulate a pressing of key with anothe scan-code (keycode).

From Pasca's notes:
> The main argument is: keycode. Its value must be a name (not a numeric code!) of **keycode** as it defined in **xkb_keycodes** type file. 
> Two other arguments serves for specifying the set of modifiers. Their names are **clearmodifiers ** and **modifiers**. The first one describes modifiers that must be cleaned from the current modifiers set and the second one describes modifiers must be added.

For example, in `mau:mau_keys`snipped, the `e`key (AD03) with the level3 modifier (_Caps_) gets redirected to the keycode FK04 (F4 key), however, the Lock modifier is cleared and the Alt modifier is "inserted".
```
key <AD03> {  // Caps + E simulates Alt + F4
        symbols[Group1]= [	  	e,		E,		F4	],
        actions[Group1]= [      NoAction(),      NoAction(),   RedirectKey(keycode=<FK04>, clearmods=Lock, modifiers=Alt) ]
        };
```

4. Create a new option in the file `xkb/rules/evdev`. Locate the section `! option = symbols` and add:
```
    ! option    =   symbols
      mau:mau_keys  =   +mau(mau_keys)
```

5. Create an option description in `xkb/rules/evdev.lst` under `! option` (e.g. right before `ctrl`):

```
mau                Custom keys for Mau
mau:mau_keys	   Shortcuts for arrows, select all, cut, copy, paste, closing windows and more 
```

6. Apply the newly created partial symbols file `mau`: 
```
setxkbmap -layout us -option mau:mau_keys
```
The `setxkbmap` command above will active the new configuration, however, it will not persist when the system is rebooted. To make the xkb_smbols file persistent, see section **Persistent xkb options in Gnome** below.

#### Persistent xkb options in Gnome
In Gnome, setting added with `setxkbmap` will be lost between sessions. Therefore it is necessary to apply the seetings via `dconf` (or `gsettings` in terminal) e.g. add `'bksp:bksp_escape'` to the _org​>gnome​>desktop​>input-sources​>xkb-options_ key (note that in `dconf` values are separated by comma+space).

### Recomended readings:
- [An Unreliable Guide to XKB Configuration](https://www.charvolant.org/doug/xkb/html/xkb.html) by Doug Palmer
- [Ivan Pascal's notes on X Keyboard Extension](https://web.archive.org/web/20070821114007/http://pascal.tsu.ru:80/en/xkb/) (using WayBackMachine Archive)
- xkeyboard-config project repository, [README.enhancing](https://gitlab.freedesktop.org/xkeyboard-config/xkeyboard-config/-/blob/master/docs/README.enhancing) documentation.