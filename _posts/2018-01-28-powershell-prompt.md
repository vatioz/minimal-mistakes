---
title: Powershell prompt
tags: powershell
category: programming
publish: true
draft: true
---

I want to share my basic powershell tweak. I don't like default powershell prompt especially because if you are deep in some folder structure, your cursor line will be filled with long path information you mostly don't need to see. 

In powershell you can add special `prompt` function to your profile (file that is run each time you start powershell) and in this function you can change what will the prompt look like. 

## Location

So my prompt simply shortens the path. So instead of 

```shell
C:\Windows\system32\WindowsPowershel\v1.0\Modules > 
```

you will see

```shell
...\Modules >
```

Or in case you are somewhere near the root

```shell
C:\Windows >
```

The prompt will always show only one last level of path (possibly with drive letter). That's huge space saving. And it is enough in most cases. And should you find yourself in the need of knowing of full path, just type `gl` or `Get-Location` or even just `dir` or `ls`. All of them will print out your current location.

## Time

Other thing I like to do is to include time in the prompt. It is rough but also very cheap way to see how long the last command took. It doesn't depend on the currently running command so you don't need to worry about any performance hit. 

One drawback is that you don't get any meaningful duration for commands after a pause - you will get time of the end of the current command, but the previous time can be way off. But in a series of commands this is good estimate.

On the other hand you will get simple history preview of your commands throughout a day.

```shell
8:15:45 C:\Windows > gl
Path
----
C:\Windows
8:15:46

```

## Code

Ok, so you may like it, how to achieve this?

```powershell
function prompt
{
  // comming soon
}
```



