---
layout: post
title:  "Recording Genode screencasts #3"
date:   2016-02-20
author: Martin
categories: genode screencast avconv
---

It's alive! Thursday I was able to record the first real screencast with my
Genode screencast subsystem. It shows Genode running Linux in a VirtualBox VM,
a CPU load monitor and the GNU-software runtime of Genode called Noux.

{% include youtube.html id="QeBvog_ZoXM" %}

The window that is initially focussed is the CLI monitor of Genode with which I
started all the other subsystems by command. Next the focus switches to the
Noux subsystem. It's running 'tail -f' on the system log file and thereby shows
also the log output of the screencast subsystem. The CPU load monitor at the
left side shows that the screencast, as configured, is recorded using the 3rd
core while the other Genode components run at the primary core. The screencast
output is stored in a folder that gets shared with the virtual machine Linux.
In Linux, the folder is mounted and shows the output file. Capturing audio
also is not supported yet.

There were two major problems I had to solve this time. First, I had to find a
way of validating the *avconv* output and second, I wanted to switch from the
current Qemu *eglgears* scenario, that was pretty slow and not that
sophisticated, to the more real-world-ish Desktop scenario mentioned above.

To get me a picture of the *avconv* output, I replaced the RAM file system as
output destination by an Ext2 file system on a persistent storage. This wasn't
a big deal. All I had to do was re-routing the corresponding session request of
the Libc VFS plugin in the *avconv* component to a new instance of the ported
Rump file-system server. The used session interface remains the same. Then I
added an instance of the AHCI driver as server for the Block session that the
Rump FS server takes as input:

~~~
<start name="ahci_drv">
	<config ata="yes">
		<policy label="rump_fs" device="0"/>
	</config>
	...
</start>
<start name="rump_fs">
	<config fs="ext2fs">
		<policy label="avconv -> rump_fs" root="/" writeable="yes"/>
	</config>
	<route>
		<service name="Block"><child name="ahci_drv"/></service>
		...
	</route>
	...
</start>
<start name="avconv">
	<config>
		...
		<arg value="rump_fs/mediafile.avi"/>
		<libc ...>
			<vfs writable="yes">
				<dir name="rump_fs"><fs label="rump_fs"/></dir>
				...
			</vfs>
		</libc>
	</config>
	<route>
		<service name="File_system"><child name="rump_fs"/></service>
		...
	</route>
	...
</start>
~~~

After that, I could mount the underlying medium on my host system when the
recording had finished and play the result in Linux. Not that there would be no
way to do this in Genode but for now its quite convenient and the persistent
storage would have been an issue anyway.

Now, I looked at the very first fruit of my hard work: A video that was not
only playable but also showed three colored gears in a window frame.  If only
it had have more action. Apparently, *avconv* recorded one and the same frame
over and over. This met well with the observation that the read operation was
issued only once at the LXFB file. Looking deeper into why the file was read
only once, it turned out that the this single *read* came from Genodes
implementation of *mmap* and not, as I thought, directly from *avconv*. As it
normally would bring a rat tail of problems when actually mapping the backing
store in *mmap*, Genode returns a mere copy of the content instead. This
explained why the frame buffer appeared static to *avconv*.

So, let's make *avconv* happy by creating a proper *mmap* implementation. In
this specific use case were *mmap* can map the Dataspace to an arbitrary
address without special requirements it should be fine. The default
implementation of the Libc VFS *mmap* is located in `libc / vfs_plugin.cc`. As
I wanted to re-use some stuff of the default implementation and also some Libc
stuff, I wanted to stay in `libc / vfs_plugin.cc` with my implementation. Thus,
I modified the default *mmap* in a way that it asks the file system which
flavour to use and in case of the "direct mapping" flavor also for the targeted
Dataspace.

With the new *mmap* in place, my video started becoming a video. A second frame
showed up and a third! For someone with very slow reactions this could already
be an action movie. The bottleneck was obvious as I still recorded
software-rendered 3D in Qemu. So, now was the time for my Desktop Genode, the
so-called Turmvilla scenario. Moving my components and configuration to
Turmvilla was a little bit more tricky than I thought. First, the Intel FB
driver had a problem with my FB splitter component. A friend gave me a good
hint to use the VESA driver instead and it worked indeed. For now I can live
with the smaller feature set of this driver but in the long term I have to fix
that.

Second, I wanted *avconv* to run on a dedicated CPU to assure a constant frame
rate even when recording a scenario with heavy work load. This worked out of
the box. I splitted the affinity space of *init* in two sub spaces and
restricted avconv to the second one:

~~~
<affinity-space width="2" />
<start name="avconv">
	<affinity xpos="1" width="1" />
	...
</start>
~~~

The last thing was to pack my screencast into a subsystem that can be started
dynamically through Genodes CLI monitor. The session routing rules for *avconv*
had the be moved to the CLI monitor and selected through their labels. I also
added a rule that routes the log output to a server component named *log*. This
component gathers most of the log output of my Turmvilla scenario and writes it
to a file.

~~~
<start name="cli_monitor">
	<route>
		<service name="File_system" label="screencast -> avconv -> rump_fs">
			<child name="rump_fs"/>
		</service>
		<service name="Framebuffer" label="screencast -> avconv">
			<child name="fb_splitter"/>
		</service>
		<service name="LOG" label="screencast -> avconv">
			<child name="log"/>
		</service>
		...
	</route>
	...
</start>
~~~

The new `subsystems / screencast.subsystem` file starts a new *init* instance
for *avconv*. This enables me to apply my affinity configuration to this single
subsystem.

~~~
<subsystem name="screencast" help="Record frame buffer">
	<binary name="init"/>
	<config>
		<parent-provides>
			<service name="File_system"/>
			<service name="Framebuffer"/>
			...
		</parent-provides>
		<affinity-space width="2" />
		...
		<start name="avconv">
			<affinity xpos="1" width="1" />
			...
		</start>
	</config>
</subsystem>
~~~

Now I can issue `start`<wbr>`screencast` in the CLI monitor and as intended, the
log output shows *avconv* starting and the CPU load monitor gives evidence that
it is running on the 3rd of my four cores. Currently, with one of my Intel-i5
cores and a resolution of 1280x800, I'm reaching around 25 FPS. The next
issues on my road map are clean dynamic termination of *avconv* (currently I'm
merely using the `-t` parameter) and support for audio recording using the
`Audio_in` session of the audio driver.

{% include abbreviations.markdown %}
