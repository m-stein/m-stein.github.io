---
layout: post
title:  "Recording Genode screencasts #2"
date:   2016-02-13
author: Martin
categories: genode screencast avconv
---

This week was pretty productive for my goal to do screencasts with a Genode
component. My ported *avconv* is now able to transcode input data from the
Dataspace of a Framebuffer session into an *mpeg4* video. Or, at least, that’s
what it tells me because for now, I’m not able to validate the output.

But let me start the story from the beginning. To replace my previous dummy
media-file input with actual frame-buffer data, I decided to use the *fbdev*
input format of *avconv*. It allows *avconv* to access frame-buffer data
through the read and write operations of a Linux character-device file.
Fortunately, Genodes Libc VFS-plugin provides an easy way for creating and
integrating custom file-system back-ends. An attractive opportunity.

But before this, I had to build *avconv* with *fbdev* support – meaning with
the *avdevice* library – first . This can be achieved by adapting the
`libav / config.*` files in the Genode sources. I configured *avdevice* in a
way that it merely brings *fbdev* support. This way, the external dependencies
were kept at a minimum. Then, as a starting point for my VFS back-end, I
created a copy of the
`vfs / log_file_system.h` header named `vfs / lxfb_file_system.h`. I stripped
it down to the mere declarations and added some statements in
`vfs / file_system_factory.cc` to make my new LXFB file-system known to the VFS
plugin. After that, I added an LXFB file to my `avconv.run` that uses the back
end:

~~~
<start name="avconv">
	...
	<config>
		...
		<arg value="-f"/> <arg value="fbdev"/>
		<arg value="-r"/> <arg value="10"/>
		<arg value="-i"/> <arg value="dev/lxfb"/>
		...
		<libc stdout="/dev/log" stderr="/dev/log">
			<vfs writable="yes">
				<dir name="dev"> <lxfb/> <log/> </dir>
				...
			</vfs>
		</libc>
	</config>
	...
</start>
~~~

Compiling this setup, I figured out that *avconv*, beside the read and write
operations, also required two IOCTLs to read the configuration of the frame
buffer, the so called Screeninfo. Therefore, it asked for a header that
declares the corresponding data structures and IOCTL names. Naive as I am, I
tried to import an appropriate header from my Linux. I quickly realized, that
this is a bad idea. Like opening the Pandora’s box of dependencies. So,
instead, I created my own header and took from Linux only what is necessary for
*avconv* by doing some try-and-error:

~~~
enum {
	FBIOGET_VSCREENINFO = 17920,
	FBIOGET_FSCREENINFO = 17922,
};
struct fb_bitfield       { /* some integers */ };
struct fb_var_screeninfo { /* some integers and fb_bitfields */ };
struct fb_fix_screeninfo { /* some integers */ };
~~~

Now the setup compiled and I got a nice error about an unsupported IOCTL beeing
issued on the LXFB file.

To implement the requested IOCTLs, I overwrote the *ioctl* method in my
`Vfs:: Lxfb_file_system` class. For the time being, I merely returned some
hard-coded Screeninfo that I took from a dump of my self-built Linux *avconv*.
This prototype then had to be connected to the VFS IOCTL mechanism through some
additions in `libc / vfs_plugin.cc`. Now, *avconv* received the hard-coded
Screeninfo and tried a first read request. But as my read implementation
returned an error by then, the program ended up in a failing *mmap* attempt
that seemed to be some kind of fall-back strategy.

As a next step, I wanted to add some real input to my scenario. I found that
the `libports / run / eglgears.run` demo might provide a good setup. It’s small
and creates dynamic frame-buffer output. So, I integrated the essentials of
that setup into my `avconv.run` script. Next, I placed the frame-buffer
splitter component, that I wrote last week, in front of the Framebuffer
driver to multiplex the Framebuffer service between Nitpicker and *avconv*.

After that, I was able to read the real Screeninfo in my *avconv* IOCTLs by
using the *mode* method of the Framebuffer session. At this point, the
*avconv* surprisingly complained about an unsupported pixel format. It turned
out that, in contrast to the *libav* main-line version, the version used by
Genode didn’t support the RGB565 format for *fbdev* input. Genodes Framebuffer
service, at the other hand, doesn’t support other pixel formats than RGB565
yet.

Fortunately, fixing this problem was easier than thought. I could extract the
commit for RGB565 support from the git history of my Linux *libav* repository
and it worked out of the box. By the way, I added all *libav* patches that I
created during my screencast work to the PATCHES variable in the `libav.port`
file. This way, they get applied automatically when doing
`tool / prepare_port``libav`.

Now, I implemented the missing read operation. Requesting the Framebuffer
Dataspace through the Framebuffer session, attaching it locally, then applying
the requested seek offset, and copying to the out buffer until the count hits
zero. Straightforward.

Now, all looked fine. Although only one read request was issued at the LXFB
file yet, *avconv* started transcoding like back when I used the media file as
input.  The last thing for this week was to validate the output. Maybe the file
can be played while it is still recorded, I thought. So, from the `avplay.run`
script, I took all configuration that was needed to run an instance of *avplay*
into my scenario. Then I replaced the media-file input of *avplay* with the
*ram_fs* file that *avconv* creates as ouput…

~~~
<start name="avplay">
	<config>
		...
		<arg value="ram/mediafile.avi"/>
		<libc stdout="/dev/log" stderr="/dev/log">
			<vfs>
				<dir name="ram"> <fs label="ram" /> </dir>
				...
			</vfs>
		</libc>
	</config>
	...
</start>
~~~

… and gave it a try. Indeed, *avplay* was able to read the file! Well, but
immediately it complained like
`ram / mediafile.avi:`<wbr>`Invalid`<wbr>`data`<wbr>`found`<wbr>`when`<wbr>`processing`<wbr>`input` and showed me a
black screen. But I’m sure that this can be fixed too… next week!

{% include abbreviations.markdown %}
