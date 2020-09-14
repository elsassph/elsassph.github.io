---
layout: post
title: "Adopting PowerShell (on Mac)"
date: "2020-01-05"
---

Oh PowerShell, I remember as a Windows user that I always avoided you like the plague. Even when it became the default terminal, since version 3 or 4, I would do everything to re-configure the old `cmd` by default. Mind you, I've always been a heavy terminal user, but somehow I happily alternated between the old `cmd` and `bash`, and didn't want to try learning the thing.

Well here I was at the end of 2019, setting up some Automated Deployment in Octopus Deploy where the main scripting language we use is PowerShell. So 13 years after its inception, here am I, using _the PoSH_ as my default shell on a Mac...

![Such power](/images/power-shell.jpg)

## PowerShell is modern

Since .NET Core happened, it has been ported to run on Mac and Linux as the open-source [PowerShell Core](https://github.com/powershell/powershell) project, which makes it a rare truly cross-platform shell. PowerShell "Core" is now also recommended on Windows.

What makes it modern:

- PowerShell pipes objects, not just stupid text you have to parse like an animal,
- PowerShell has good, readable, [documentation and learning material](https://docs.microsoft.com/en-us/powershell/scripting/learn/understanding-important-powershell-concepts?view=powershell-7),
- PowerShell has a package manager to easily install modules extending the shell functionalities,
- PowerShell scripts and modules can be properly unit-tested: [Pester](https://github.com/Pester/Pester) seem to be the state of the art module, and it works impressively well (commands mocking, rich assertions),
- And VS Code has [first-class support for PowerShell](https://code.visualstudio.com/docs/languages/powershell) - code completion and all!

Piping objects is extra powerful and PowerShell has first-class support for:

- Creating, manipulating, iterating all sorts of lists and objects,
- File objects, with properties like `Path`, `Name`, `Size`,...
- File content as collection of lines,
- JSON parsing/serialising (`ConvertFrom-Json`, `ConvertTo-Json`),
- Markdown, CSV, HTML parsing/serialising/rendering,
- Calling REST APIs (`Invoke-RestMethod`),
- etc.

PS: this all is NOT case-sensitive.

## Show me some code!

Doing something with files:

```powershell
# inspect properties and methods of (non directory) file objects
Get-ChildItem -Attributes !Directory | Get-Member

# short version using aliases and parameters shortening
dir -at !D | gm

# list filenames without extensions (% is an alias for `Foreach-Object`)
dir *.png | % { $_.BaseName }

# same but numbered
dir *.png | % { $i=0 } { "$i : $($_.BaseName)"; $i++ }
```

Extracting info from a `package.json`:

```powershell
# read and convert package.json
$pkg = Get-Content ./package.json | ConvertFrom-Json

# print the version
$pkg.version

# print the npm scripts as a table "name : value"
$pkg.scripts

# extract the script names
$pkg.scripts.PSObject.Properties | % { $_.Name }
```

## Can it be your default shell?

Because [why not?](https://sqlsunday.com/2019/03/04/how-to-set-up-a-beautiful-powershell-core-terminal-on-mac-os/) (also you're a shell power-user)

### It won't break your habits (too much)

After 20-something years using unix shells I'd have a hard time stopping using `ls`, `mv`, `rm`, etc (I even use them on Windows, thanks to _msys-git_).

Good news is that those commands don't stop working: binaries, and shebang scripts still work as expected, so your collection of painfully written Bash scripts will continue to serve.

### Tips and tricks

#### "Where is my PowerShell profile?"

Similarly to other shells, a `ps1` file is executed in logged-in sessions. The file can be found using the `$PROFILE` variable which points (on Mac) to:

```
 /Users/<username>/.config/powershell/Microsoft.PowerShell_profile.ps1
```

**Hot tips:**

- Any **variable** or **function** (also called "CmdLets") declared in this script will be available during the session,
- PowerShell language is **entirely case insensitive**,
- There are default **aliases** for most of the common CmdLets:
    type `gal` (e.g. `Get-Alias`) for the list.

#### "How do I manage the PATH here?"

Since PowerShell 7, the Mac's usual `/etc/paths` is automatically loaded. Otherwise you can add to the `PATH` by changing it in the `Env` drive\*:

```powershell
# add to the PATH (':' on Mac/Linux, ';' on Win)
$Env:PATH += ":/opt/some_tool/bin"
```

_\***drive**?!_ Yes, like a virtual disk, you can `dir env:` to list environment variables (exceptionally case-sensitive), and set things like `$Env:JAVA_HOME="/path/to/java/"`.

See also `Function:`, `Variable:`, `Alias:`, `Temp:`. You can unset a variable `$foo` using `del variable:foo`. Crazy times, uh?

#### "I want a fancy git-aware prompt like with zsh"

**Edit:** I'd simply recommend [starship.rs](https://starship.rs/), a smart and fast cross-shells prompt.

I previously used a PS-native module for that: [posh-git](https://github.com/dahlbyk/posh-git)

#### "How do I manage Node versions?"

There's also a module for that! [ps-nvm](https://github.com/aaronpowell/ps-nvm)

You'll need to load the module and set your preferred version in your `profile.ps1`:

```powershell
# load module
Import-Module nvm
# set default version
Set-NodeVersion 12
```

_I've also contributed a PR in order to be able to be able to_ [_persist the default node version on Mac/Linux_](https://github.com/aaronpowell/ps-nvm/pull/86) _like nvm-sh but it works well like that._

#### "How do I pass environment variables to commands?"

You can't `FLAG=true yarn dev` to set environment variables when calling a process. But like on Windows you can use [cross-env](https://www.npmjs.com/package/cross-env) for that:

```powershell
npm install cross-env -g
cross-env FLAG=true yarn dev
```

#### "How do I _sudo_ something?"

This one doesn't have a direct equivalent. Like on Windows you would generally escalate your session: `sudo pwsh -NoProfile` will spawn an admin-level session which you need to `exit` when you're done.

#### "Is there a commands history?"

Sure, thanks to the pre-installed [PSReadLine](https://docs.microsoft.com/en-us/powershell/module/psreadline/?view=powershell-7) module:

- _Up / Down_, unsurprisingly, moves through history (ignoring anything you may have already typed),
- _Ctrl+R_ initiates search: type something to search (anywhere) in a previous command; then _Ctrl+R / Ctrl+S_ move backward/forward in the results.

#### "How do I call multiple commands?"

Use `;` or `&&` (since PowerShell 7) to call several commands one after the other (`yarn run clean ; yarn dev`). Note: when using `;` the commands don't stop if one fails.

It's sometimes usefull to be able to run multiple, parametric commands for tool:

```powershell
# execute a list of commands - stop if one fails
function Invoke-StopOnError($Cmds) {
  foreach($Cmd in $Cmds) {
    Invoke-Expression $Cmd
    if ($LASTEXITCODE -ne 0) { return }
  }
}

# multi-step command: stash work, switch to branch, pull, pop stash
function StashTo($Branch) {
  Invoke-StopOnError @(
    "git stash",
    "git checkout $Branch",
    "git pull",
    "git stash pop"
  )
}
```

## Famous last words

I've run this shell for a few months on a daily basis at work. I'm already able to write much more complex scripts compared to Bash which I don't bother fighting with unless it's for trivial matter.

Overall the experience on Mac isn't very different from using a shell which is neither `bash` nor `zsh` - once you've solved a few initial oddities, it works really well. Ultimately it's fun, and rewarding, to get out of your comfort zone!

[Your turn now](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7) ;)

### Edit: 10 months later

I'm still very much using PowerShell on Mac and Win. PowerShell 7 is even friendlieer for new starters.

One thing I changed: I'm now using the cross-shell, smart and fast, [starship.rs](https://starship.rs/) as custom prompt, because it has much more features than posh-git.
