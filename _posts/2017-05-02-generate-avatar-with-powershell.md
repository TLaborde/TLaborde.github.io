---
layout: post
title: "Generating Anime Avatar with Powershell (and some flash game)"
description: "Module generating a unique avatar using a webBrowser and a lot of stupid shortcut"
modified: 2017-05-02
tags: [powershell idea webbrowser generate]
---

## Context
The silly idea of [OS-tan](https://en.wikipedia.org/wiki/OS-tan), the personification of OS as cute manga characters made me think of a sillier idea: show server monitoring as anime girl. Let's write down the requirements.

### Requirements
* Each server/service should be represented by a *unique* avatar, like an identicon.

### Best Implementation
* Create several dozen of body part layer template on transparent background, assign for each a palette of allowed colors.
* Generate a unique hash from a service/server name.
* From the hash, select layers and color and merge them.

### Current Crappy Implementation
* Generate a unique hash from a service/server name.
* Feed the hash as a seed to a random number generator.
* Open a Flash game to create your own anime character, and click (pseudo-)randomly.
* Do a screenshot.

## Part 1: Generate a unique seed from a string

This part will be the sanest part of the final result. We will use .Net to generate a MD5 hash of a string, then truncate it from an int128 to an int32, required to seed ```Get-Random``` in PowerShell. MD5 shouldn't be used as a hash to protect data, but for our purpose, which is to generate scattered numbers from an input of a small size, it's good enough. Knuth probably would recommend something more robust, but it's good enough for this first step. Now, the code:

{% highlight PowerShell %}
function Get-SeedFromString([String] $String) { 
    $bytes = [System.Security.Cryptography.HashAlgorithm]::Create("MD5").ComputeHash([System.Text.Encoding]::UTF8.GetBytes($String))
    [bitconverter]::ToInt32($bytes,0)
}
{% endhighlight %}

## Part 2: Feed the hash as a seed to a random number generator.

This part is very easy, we just need to set ```Get-Random```.

{% highlight PowerShell %}
$seed = Get-SeedFromString $name
Get-Random -SetSeed $seed
{% endhighlight %}

Every times ```SetSeed``` is called, it will reset the generator, which will ensure that our process can be reproduced.

## Part 4: Do a screenshot.

Wait, why is Part 4 before Part 3? Because while we start getting dirty, it's not full madness yet, and it's short. We will write a function doing a screenshot of a rectangular area of the screen, needing the coordinate of the top right corner and the bottom left corner.

{% highlight PowerShell %}
function Get-ScreenShot([Int]$TopRightX,[Int]$TopRightY,[Int]$BottomLeftX,[Int]$BottomLeftY, [string]$name) {
    $bounds = [Drawing.Rectangle]::FromLTRB($TopRightX,$TopRightY, $BottomLeftX,$BottomLeftY)
    $bmp = New-Object Drawing.Bitmap $bounds.width, $bounds.height
    $graphics = [Drawing.Graphics]::FromImage($bmp)
    $graphics.CopyFromScreen($bounds.Location, [Drawing.Point]::Empty, $bounds.size)

    $screenCapturePathBase = "$pwd\$name.png"
    if (Test-Path $screenCapturePathBase) {
        Remove-Item $screenCapturePathBase
    }
    $bmp.Save($screenCapturePathBase)
    $graphics.Dispose()
    $bmp.Dispose()
}
{% endhighlight %}

A proper screenshot function would check if the file does not exist, and maybe increment the name. For our goal, we don't need that.

## Part 3: Open a Flash game to create your own anime character, and click (pseudo-)randomly.

The madness really starts now. Since I have not an artsy bone in my coder body, I will rely on more talented people. I tested a lot of Anime Avatar Generator games and finally decided to work with one of the latest fron [Gen-8](http://gen8.deviantart.com/art/Anime-Face-Maker-2-182829244). The menus are relatively consistant, as is the color palette. We need to do several thing:
* Package the swf to open it in a webbrowser controlled by Powershell
* Click on the "play button" once the swf is loaded
* Click around a bunch of time
* Screenshot and quit

### Helper function: Click somewhere on the screen

I just used some class copy/pasted from Internet, it create a static method to click somewhere. I think it's buggy for weird multi screen setup, but we are not trying to do thing in the proper way, I think it's now clear.

{% highlight PowerShell %}
$cSource = @'
using System;
using System.Drawing;
using System.Runtime.InteropServices;
using System.Windows.Forms;
public class Clicker
{
//https://msdn.microsoft.com/en-us/library/windows/desktop/ms646270(v=vs.85).aspx
[StructLayout(LayoutKind.Sequential)]
struct INPUT
{ 
    public int        type; // 0 = INPUT_MOUSE,
                            // 1 = INPUT_KEYBOARD
                            // 2 = INPUT_HARDWARE
    public MOUSEINPUT mi;
}

//https://msdn.microsoft.com/en-us/library/windows/desktop/ms646273(v=vs.85).aspx
[StructLayout(LayoutKind.Sequential)]
struct MOUSEINPUT
{
    public int    dx ;
    public int    dy ;
    public int    mouseData ;
    public int    dwFlags;
    public int    time;
    public IntPtr dwExtraInfo;
}

//This covers most use cases although complex mice may have additional buttons
//There are additional constants you can use for those cases, see the msdn page
const int MOUSEEVENTF_MOVED      = 0x0001 ;
const int MOUSEEVENTF_LEFTDOWN   = 0x0002 ;
const int MOUSEEVENTF_LEFTUP     = 0x0004 ;
const int MOUSEEVENTF_RIGHTDOWN  = 0x0008 ;
const int MOUSEEVENTF_RIGHTUP    = 0x0010 ;
const int MOUSEEVENTF_MIDDLEDOWN = 0x0020 ;
const int MOUSEEVENTF_MIDDLEUP   = 0x0040 ;
const int MOUSEEVENTF_WHEEL      = 0x0080 ;
const int MOUSEEVENTF_XDOWN      = 0x0100 ;
const int MOUSEEVENTF_XUP        = 0x0200 ;
const int MOUSEEVENTF_ABSOLUTE   = 0x8000 ;

const int screen_length = 0x10000 ;

//https://msdn.microsoft.com/en-us/library/windows/desktop/ms646310(v=vs.85).aspx
[System.Runtime.InteropServices.DllImport("user32.dll")]
extern static uint SendInput(uint nInputs, INPUT[] pInputs, int cbSize);

public static void LeftClickAtPoint(int x, int y)
{
    //Move the mouse
    INPUT[] input = new INPUT[3];
    input[0].mi.dx = x*(65535/System.Windows.Forms.Screen.PrimaryScreen.Bounds.Width);
    input[0].mi.dy = y*(65535/System.Windows.Forms.Screen.PrimaryScreen.Bounds.Height);
    input[0].mi.dwFlags = MOUSEEVENTF_MOVED | MOUSEEVENTF_ABSOLUTE;
    //Left mouse button down
    input[1].mi.dwFlags = MOUSEEVENTF_LEFTDOWN;
    //Left mouse button up
    input[2].mi.dwFlags = MOUSEEVENTF_LEFTUP;
    SendInput(3, input, Marshal.SizeOf(input[0]));
}
}
'@
Add-Type -TypeDefinition $cSource -ReferencedAssemblies System.Windows.Forms,System.Drawing
{% endhighlight %}

### Packaging the Flash game to be open by PowerShell

We will use a Form with a WebBrowser object to manipulate the flash generator. For that, we need a html page wrapper, since we cannot just inline a .SWF file. Quick and dirty ```generator.html``` content:

```html
<object>
    <embed src="anime_face_maker_2_by_gen8-d30uny4.swf" width="900" height="650"></embed>
</object>
```

### Clicking around

We are really in the "meat" of the generator. Let's say we have opened the window, clicked on "play" and that the game is fully loaded. We will simply use the ```Get-Random``` function to generate coordinate where to click inside the window. To get more colorful picture, we will click first in the menus/feature selections, then in the palette. The proper way would be to define the clicking zone relative to the window, but since we force the position of the window, we will deal only in absolute (like a Sith).

{% highlight PowerShell %}
function Invoke-FeatureGenerator{
    1..200 | % {
        $x = 300..940 | Get-Random
        $y = 120..615 | Get-Random
        [Clicker]::LeftClickAtPoint($x,$y)
        # force pickup color
        $x = 50..277 | Get-Random
        $y = 468..598 | Get-Random
        [Clicker]::LeftClickAtPoint($x,$y)
    }
}
{% endhighlight %}

Remember, since we forced a seed just before, we can replay the script and it will click at the same place in the same order. We loop 200 hundred times, because why not. Really, 50 would do too, but the difference of speed is negligible (since most of the is spent waiting for the windows and the SWF to load).

## Putting it all together

The main function will create a form, add the webpage, click on "Play", click (pseudo-)randomly around, do a screenshot, and close the form. The interesting part is the use of timers to have sequential actions. Each timer' trigger unregisters itself (so it runs only once) and prepare the next one. Since the code is a bit long and full of boiler plate, here is just the overall gist, that can be run directly.

{% gist TLaborde/60af3ef70d8b17f6e05e5ed62eaca66f %}

Once you have run the content of the gist once, you will have in the current folder the HTML template and the SWF file. You can then just run the command ```Generate-Avatar "PowerShell"``` and check the result in the current directory.

<figure>
	<img src="/images/PowerShell.png" alt="">
	<figcaption>Result for <code class="highlighter-rouge">Generate-Avatar "PowerShell"</code></figcaption>
</figure>

## Going further

I'm on the fence, either I make this script into a proper Module, or I just burn it down to the ground. I may want to also add a function to generate different expression, like a sleeping face (when the server is on maintenance) or anger (when the monitoring is getting errors).