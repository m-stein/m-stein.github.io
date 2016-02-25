---
layout: post
title:  "Recording Genode screencasts #1"
date:   2016-02-06
author: Martin
categories: genode screencast avconv
---

Since Monday, I have a new project! I'm going to write a Genode subsystem that
can be used to record screencasts. It was in the train from the FOSDEM 2016
back to Dresden when I decided to attend to this topic. I sat next to Norman
and we talked about an [online
discussion](https://github.com/genodelabs/genode/issues/1000#issuecomment-37790271)
that addressed the future of Genode's live images and had recently received
some stirring input. The conclusion of the discussion was that the live-image
approach are the wrong way to go. For a reliably demonstration of new Genode
features to a broad public, this was the credo, screencasts might be a better
solution. Well, I had just finished my previous project about [Genode as
USB-Armory hypervisor](http://genode.org/documentation/articles/usb_armory) and
felt that I could need a break from such low-level programming. So, let's go!

I knew that my screencast solution should take a regular Framebuffer session as
input as this promises most flexibility in common Genode setups. To connect to
a frame buffer that is used by another component, I saw two options. Either my
recording component is interposed between Framebuffer server and client or the
server provides multiple sessions to the same frame buffer. The former approach
offers more potential for optimization as the refresh signals of the client can
be intercepted. The latter approach gives me more flexibility regarding the
location of the recording component. The decision came with the idea to port
the Libav tool *avconv* for the recording task. This tool promised to be a
low hanging fruit as most of its library dependencies were already ported but
it also ruled out the interposing approach for me. When porting *avconv*,
leaving it the POSIX program it is, it would have been a bottomless pit to hack
in a Genode server role also.

These thoughts led me to my first task for this week. Because most frame-buffer
drivers do not support multiple clients, an additional component, called FB
splitter, was needed to act as a multiplexer. In principal, this FB splitter,
located in `server/ fb_splitter`, merely forwards requests to the original
server. By using one Entrypoint for all FB-splitter clients, the requests
automatically get serialized. The splitter must currently also store a copy of
all Capabilities that get delegated by him. This avoids troubles with the
garbage-collection manners of kernels like NOVA or seL4. Apart from that little
quirk, the splitter worked out of the box:

~~~xml
<start name="fb_splitter">
	<provides><service name="Framebuffer"/></provides>
	<route>
		<service name="Framebuffer"> <child name="fb_drv"/> </service>
		...
	</route>
	...
</start>
<start name="nitpicker">
	<route>
		<service name="Framebuffer"><child name="fb_splitter"/></service>
			...
	</route>
	...
</start>
~~~

The port of *avconv* was also pretty straightforward. Initially, I added only
the core unit `avconv.c`. Then, I successively resolved the compilation errors.
Mostly they were about missing dependencies that could be resolved by adding
libraries, other sources, or include paths. The *avplay* port was a good source
of inspiration during this process.

~~~make
include $(REP_DIR)/lib/import/import-av.inc

TARGET    = avconv                   # how the component shall be called
CC_C_OPT += -DLIBAV_VERSION=\"0.8\"  # select supported libav version
SRC_C    += avconv.c cmdutils.c
LIBS     += avfilter avformat avcodec avutil swscale libc libm config_args
INC_DIR  += $(PRG_DIR) $(REP_DIR)/src/lib/libav

vpath %.c $(call select_from_ports,libav)/src/lib/libav
~~~

The first step I wanted to take with *avconv* was transcoding a small video
without changing the codec. I created a new test script `avconv.run` that
starts a timer driver, a RAM file-system server, and, of course, *avconv*:

~~~xml
<start name="timer">...</start>
<start name="ram_fs">
	<provides><service name="File_system"/></provides>
	<config>
		<policy label="avconv -> ram" root="/" writeable="yes"/>
	</config>
	...
</start>
<start name="avconv">
	<config>
		<arg value="avconv"/>
		<arg value="-i"/>   <arg value="mediafile"/>
		<arg value="-an"/>
		<arg value="-c:v"/> <arg value="mpeg4"/>
		<arg value="ram/mediafile.avi"/>
		<libc stdout="/dev/log" stderr="/dev/log">
			<vfs writable="yes">
				<dir name="dev"> <log/>            </dir>
				<dir name="ram"> <fs label="ram"/> </dir>
				<rom name="mediafile"/>
			</vfs>
		</libc>
	</config>
	<route>
		<service name="File_system">
			<if-arg key="label" value="ram"/><child name="ram_fs"/>
		</service>
		...
	</route>
	...
</start>
~~~

In this test, the input media file, as one of the boot modules, is available
through the ROM service of Genodes Core while the RAM FS server provides some
space for writing back the output of *avconv*. Using Genodes `config_args`
library I was able to define the arguments for *avconv* in its configuration
ROM. I dropped the audio via `-an` for now to simplify things. Through
`-c:v`<wbr>`mpeg4`, I set the ouput codec to that of the input file. Last but
not least, I configured the Libc in *avconv* to provide a VFS that contains
three nodes. The *log* node is a character device that forwards the standard
output of *avconv* to a Genode log session, the *fs* node with the "ram" label
provides access to the RAM FS server and the *rom* node to the ROM session of
the input media file.

Once running, the test did surprisingly well. After some processing, the log
output of *avconv* reported success. It was Friday afternoon and I was happy
about the state that I had reached so far.

{% include abbreviations.markdown %}
