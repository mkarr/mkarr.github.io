## Reverse Engineering a Wifi Boroscope 

By: Michael Karr - 2020/06/16

### Introduction

A while ago, I purchased an inexpensive boroscope/borescope manufactured by DEPSTECH, for automotive and household use. Eventually, I wanted to use the video produced by this camera outside the mobile app. I needed a way to figure out how to capture the stream and manipulate it into some usable form on my main desktop PC.

### Getting Started

The first step was to packet capture the traffic using an Android packet capture app. What immediately stood out was large amounts of traffic on port 50000. Upon further inspection of this traffic, I saw JFIF magic strings appearing regularly. 

![Packet Capture 1](/boro_packet_1.png) ![Packet Capture 2](/boro_packet_2.png)


These JFIF streams seemed to jump out and scream, "Hey, MJPEG stream here!". That was easy. Thus, my next step was to attempt to play the stream using ffmpeg. However, when doing so, it was quickly apparant that something was not quite right. The stream was choppy and full of artifacts.

![Corrupt FFPlay](/boro_ffplay_corrupt.png)

### Prior Work

I decided to Google around to see if anyone else had similar ideas to me. It turns out that [Nathan Henrie](https://n8henrie.com/) and [Matthew Plough](https://mplough.github.io/) had both tackled differnt aspects of the problem. Nathan had dumped firmware images, and decoded the camera preferences and control protocol, and Matthew had begun work on repairing the MJPEG stream.

Rather than duplicate work, I cloned a copy of Matthew's work and managed to build it on my system. It was functional, however the MJPEG stream was not perfect, this I decided to look deeper into the problem.

### Reverse Engineering Firmware

Nathan had already dumped the firmware and gracously provided it for download, and Matthew demonstrated how to dump it using binwalk. Replicating this, I obtained a copy of the `app_cam` binary. These boroscopes are actually full fledged Linux machines, running a MIPS processor and a suprisingly un-stripped-down Busybox environment, it even still had vi installed. This was a good starting point, as the app was a standard ELF formatted binary.

I loaded the binary into [Ghidra](https://ghidra-sre.org/), the NSA's excellent decompilation tool, without much fuss. After the initial analysis stages, I looked for interesting symbols and exports. Among the more interesting ones, were `uvcGrab()`, `video_capture_thread()`, `video_encode_pass()`, and `video_set_headtail()`.

![Ghidra Symbols](/boro_ghidra_symbols.png)

Inspecting the code, it appears that the app runs a the `video_capture_thread()` function in a thread, repeatedly calling the functions `video_encode_pass()` and `video_set_headtail()` in a loop. Additionally there is a call to `uvcGrab()`.

![Ghidra Video Thread](/boro_ghidra_video_thread.png)

The `uvcGrab()` symbol name, in combination with some others, and some Googling lead to the [mjpg-streamer](https://sourceforge.net/projects/mjpg-streamer/) project. This appears to be a utility to create an MJPEG stream from usb webcams under Linux, something very similar to what our camera is doing. A lot of the code appears to be identical, but not exact. The code bases likely diverged at some point in the past and contined with independant development.

However, in any case, a lot of the control flow and data structures seemed the same. After filling in a bunch of struct and variable type information, and renaming a lot of variables in accordance to what they appeared to be used for when calling the mjpg-streamer and V4L2 code, I was able to identify the main video frame buffer, and size information. This is important as it is used in the `video_encode_pass()` and `video_set_headtail()` functions.

The `video_set_headtail()` function is pretty straightfoward, and Matthew already put some work into this area. It inserts a header and footer around the frame buffer and the start and end frame tags. This header carries 12 values of various 8, 16, and 32 bit types, all repacked for little endianness. The most important of these header parameters is the size value. This is used for the next step, "encoding" MJPEG frame.

### Encoding

Immediately after the header and footer are inserted, the `video_encode_pass()` function is called. This appears to be where the juicy bits are going to happen. Or so you would thing. Upon decompilaion, you are presented with the following.

![Ghidra Video Encode](/boro_ghidra_encode.png)


Is it really that easy? Invert one byte? Turns out it is. It makes a lot of sense now, why the corruption in the frames started direcly in the center of the image.

### Final Steps

After this, I pretty much concluded my reverse engineering phase of this project. My next steps were to write a small utility to correct these frames, taking input from a file or network stream, and outputing to the same. It is written in C and should be portable to just about anything POSIX. This is currently released on my [GitHub repo](https://github.com/mkarr/boroscope_stream_fixer). I'll likely add Windows support in the near future and maybe a bit more error tolerance. 


![Video Fixed Demo](/boro_fixed.png)


---

[Home](/)
