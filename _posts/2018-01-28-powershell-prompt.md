---
title: Powershell prompt
excerpt: My basic Powershell tweak
tags: powershell
category: programming
publish: true
draft: false
---

I want to share my basic powershell tweak. I don't like default powershell prompt especially because if you are deep in some folder structure, your cursor line will be filled with long path information you mostly don't need to see. 

In powershell you can add special `prompt` function to your profile (file that is run each time you start powershell) and in this function you can change what will the prompt look like. 

## Location

First, my prompt simply shortens the path. So instead of 

```powershell
C:\Windows\system32\WindowsPowershel\v1.0\Modules > 
```

you will see

```powershell
...\Modules >
```

Or in case you are somewhere near the root

```powershell
C:\Windows >
```

The prompt will always show only one last level of path (possibly with drive letter). That's huge space saving. And it is enough in most cases. And should you find yourself in the need of knowing of full path, just type `gl` or `Get-Location` or even just `dir` or `ls`. All of them will print out your current location.

## Time

Other thing I like to do is to include time in the prompt. It is rough but also very cheap way to see how long the last command took. It doesn't depend on the currently running command so you don't need to worry about any performance hit. 

One drawback is that you don't get any meaningful duration for commands after a pause - you will get time of the end of the current command, but the previous time can be way off. But in a series of commands this is good estimate.

On the other hand you will get simple history preview of your commands throughout a day.

```shell
8:15:45 C:\Windows> gl
Path
----
C:\Windows
8:15:46 C:\Windows>

```

## Code

Ok, so how to achieve this?

```powershell
function prompt
{
	# get actual location
	$loc = gl 
	$sep = [IO.Path]::DirectorySeparatorChar
	
	# get separated parts of the path
	$parts = $loc.tostring().Split($sep) 
	
	Write-Host (get-date -f "HH:mm:ss ") -nonewline -foregroundcolor Magenta
	
	if($parts.Length -gt 2 -and $parts[1] -ne "" ) # display last part of the path
	{
		# get only the last part
		$lastpart = $parts[$parts.Length-1] 
		
		# print last part as prompt
		Write-Host ("..$sep$lastpart >") -nonewline -foregroundcolor Magenta 
	}
	elseif ($parts.Length -eq 2) # short path, so display with drive letter
	{
		Write-Host ( $parts[0] )  -nonewline -foregroundcolor Magenta # display drive letter
		if($parts[1].length -gt 0) # is there something after drive letter?
		{
			Write-host ([IO.Path]::DirectorySeparatorChar + $parts[1] )  -nonewline -foregroundcolor Magenta # display first folder after drive letter
		} 
		Write-Host (">") -nonewline -foregroundcolor Magenta # append angle bracket
	}
	else # no parts in path...
	{
		Write-Host (" >") -nonewline -foregroundcolor Magenta 
	}
	
	# this is important for some unknown reason ;)
	return " "
}
```



