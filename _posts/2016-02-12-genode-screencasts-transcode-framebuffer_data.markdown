---
layout: post
title:  "Genode screencasts - Transcode framebuffer data"
date:   2016-02-12 21:51:51 +0100
author: martin
categories: genode screencast avconv
---
Yesterday, I've added RGB565 support to the fbdev input of my old libav
version. I also implemented the read operation for my VFS LXFB file system in a
way that it reads from the Framebuffer dataspace considering the seek and
count arguments.  At the end, I started integrating avplay into my test script
to be able to check whether the output of avconv is sensible.

RGB565 was easier than thought. I extracted the one-line patch from the
history of my Linux libav repository. I used the opportunity to also clean-up
the other libav patch that I created during my screencast work. I added both
patches to the libav.port file and tested the whole thing with `prepare_port`.

For the read operation I took the `Vfs::Block_file_system read` implementation
as starting point. But, of course, reading from a single dataspace is much
easier. I could also have implemented it from scratch :smiley:

Unfortunately the freshly integrated avplay complains about `ram/mediafile.avi:
Invalid data found when processing input` now. But at least it is able to read
the file while avconv is writing it. I took most of the expect stuff for the
avplay integration from `libports/run/avplay.run` and replaced the mediafile
input with the `ram_fs` file that avconv creates.
