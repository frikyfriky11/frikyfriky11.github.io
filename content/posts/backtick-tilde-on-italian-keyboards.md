+++
date = 2023-03-22T21:21:02+01:00
title = "Backtick and tilde on an Italian keyboard"
description = "Typing the backtick and tilde characters on an Italian keyboard without needing ASCII keycodes"
authors = ["Stefano Previato"]
tags = ["backtick", "tilde", "Italian keyboard", "ASCII keycodes"]
categories = ["Windows", "Tips"]
+++

If you're using an Italian keyboard or any keyboard that does not have built-in keys for backtick \` and tilde \~ characters and you do some kind of programming or simply write your notes in Markdown everyday like me, you'll know the struggle of hitting the `Alt + 126` ASCII keycode for the tilde and `Alt + 96` for the backtick.

Everything is smooth until you need to write a code block in Markdown that needs 6 (six!) backticks in order to be recognized correctly... that means hitting `Alt + 96` six times, or 12 key strokes in total.

Even worse, if you're writing from a laptop keyboard that does not have a numpad, good luck!

It becomes really boring pretty fast, right?

That's why you absolutely need to create an AutoHotkey script for this!

- Download and install [AutoHotkey](https://www.autohotkey.com/) or just `choco install autohotkey` if you use the [Chocolatey](https://chocolatey.org/) package manager
- Go to a folder on your local drive and _Right-click -> New -> AutoHotkey script_ (you could also create a blank `.txt` file but you would need to set the encoding to _UTF-8 with BOM_ for this script to work correctly, and creating an AutoHotkey script this way does everything automatically)
- Right-click on the file you just created and click on _Edit script_
- Replace the contents with the following black-magic script (I'll explain later what it does):

```
#NoEnv  ; Recommended for performance and compatibility with future AutoHotkey releases.
; #Warn  ; Enable warnings to assist with detecting common errors.
SendMode Input  ; Recommended for new scripts due to its superior speed and reliability.
SetWorkingDir %A_ScriptDir%  ; Ensures a consistent starting directory.

; Tilde character with AltGr+ì
<^>!ì::~

; Backtick character with AltGr+'
<^>!'::`
```

- Save the file and double-click it to run it
- The script should now be running in the background (you can check it by finding the little _green H_ icon in the tray bar)
- Open a notepad and try to press the `AltGr + '` or the `AltGr + ì` key combinations
- Profit!

I decided to use the `AltGr + '` and `AltGr + ì` key combinations because they don't do anything useful on my layout, and they are pretty comfy to reach while typing.

So what is that hieroglyphic language `` <^>!'::` ``?

The caret symbol `^` represents the Ctrl key, the exclamation mark `!` represents the Alt key, and the apostrophe `'` represents itself, the `::` represent the boundary between the shortcut on the left and what needs to happen on the right, and finally the `` ` `` represents the character we want to output.

Windows represents the `AltGr` key as a combination of `Left Ctrl` and `Right Alt`, and the modifiers `<` and `>` prefixed to a symbol represent the side of the keyboard to emulate, in this case `<^` means `Left Ctrl` and `>!` means `Right Alt`.

Now that you know how it all works and you've set it up, you'll quickly fall in love with it and I'm pretty sure you'll want this script to run at every startup! To do that you can create a simple scheduled task with the _Windows Task Scheduler_:

- In the _General_ tab make sure to select _Run only when user is logged on_
- In the _Triggers_ tab make sure to create a trigger that runs _At log on_
- In the _Actions_ tab create a new _Start a program_ action, browse to your `AutoHotkey.exe` (by default it's located at `C:\Program Files\AutoHotkey\AutoHotkey.exe`) and paste the full path of your script as a an argument (to grab the full path just right-click on the script while you're pressing the `Shift` key, then select _Copy as path_)
- If you're working on a laptop, deselect the _Start the task only if the computer is on AC power_ flag from the _Conditions_ tab

\~\~\~ Now go and enjoy writing \` backtick \` and \~ tilde \~ characters all over your documents! \~\~\~
