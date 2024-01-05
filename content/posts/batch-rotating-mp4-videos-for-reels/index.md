+++ 
date = 2022-08-31T21:43:06+02:00
title = "Batch rotating multiple mp4 videos (useful for Instagram Reels)"
description = "Rotate an entire folder of multiple mp4 videos with a single PowerShell script, useful if you like to edit videos for Instagram Reels"
authors = ["Stefano Previato"]
tags = ["video", "rotating", "batch", "reels", "Instagram", "ffmpeg", "PowerShell"]
categories = ["Instagram", "Video", "PowerShell"]
+++

Whenever I have some free time (almost never these days, sigh!), you can find me shooting photos around the city. Usually I like to post the best content of a shooting day on Instagram, and while I am mostly focused on photography, I sometimes like to post some videos as Instagram Reels as well.

Instagram Reels have a 9:16 aspect ratio (1080x1920) while my camera (a Canon EOS R) shoots in 9:16 (1920x1080) and doesn't automatically rotate the videos when I shoot them portrait, unlike the photo mode that does this by default.

Whenever you load the files in Premiere Pro, they are all in landscape orientation, and rotating them to portrait is just way too much work for some fun free time activity.

Luckily, the universe brought us the [ffmpeg](https://ffmpeg.org/) tool, made by some of the most amazing developers around, and it's free to use too! It's very likely that you already have this tool installed on your machine, but if you don't have it I'd suggest taking a look at the [K-Lite Codec Pack](https://codecguide.com/download_kl.htm) project. It's a collection of the most used (and not so used) codecs and tools for video playback, and it includes `ffmpeg` as well!

Once you have installed it, you can use this little snippet of PowerShell code:

```powershell
ls -Filter *.mp4 | foreach {
  ffmpeg -i $_.FullName -metadata:s:v rotate=90 -vcodec copy -acodec copy "$($_.BaseName)-rotated.mp4"
  (Get-Item "$($_.BaseName)-rotated.mp4").CreationTime = (Get-Item $_.FullName).CreationTime
  (Get-Item "$($_.BaseName)-rotated.mp4").LastWriteTime = (Get-Item $_.FullName).LastWriteTime
}
```

Let me explain what this does:

- first, it enumerates the contents of the current folder, searching for \*.mp4 files
- for each file, it invokes the `ffmpeg` utility with the following parameters:
  - `-i $_.FullName` to indicate the input file (`$_` in a PowerShell `foreach` indicates the current item)
  - `-metadata:s:v rotate=90` sets the `rotate` metadata to `90` for all the video streams in the file
  - `-vcodec copy -acodec copy` copies all the audio and video streams as they are
  - `"$($_.BaseName)-rotated.mp4"` sets the output file name to be the same as the input name (without extension) with the added `-rotated` suffix
- next, it takes the original file `CreationTime` and `LastWriteTime` and copies them to the newly generated file

That's it! You now have the ability to rotate a bunch of video files with just a couple of keystrokes instead of doing it by hand one by one!

Have fun!
