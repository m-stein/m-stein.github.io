---
layout: post
title:  "Recording Genode screencasts #3"
date:   2016-02-20
author: Martin
categories: genode screencast avconv
---

It's alive! Thursday I was able to record a smooth screencast of my Genode
desktop setup on a Lenovo X201. One minute with 20 FPS showing Genode while
running Linux in VirtualBox, a CPU load monitor and the GNU-software runtime
Noux:

[embedded video]

There were two major problems I had to solve this time. First, I had to find a
way of validating the *avconv* output and second, I wanted to switch from the
current Qemu *eglgears* scenario, that was pretty slow and not so awesome, to
the more real-world-ish Desktop scenario mentioned above.

To get me a picture of the *avconv* output, I replaced the RAM file system as
output destination by an Ext2 file system on a persistent storage. This wasn't
a big deal. All I had to do was re-routing the corresponding session request of
the Libc VFS plugin in the *avconv* component to a new instance of the ported
Rump file-system server. The session interface remains the same. Then I added
an instance of the AHCI driver as server for the Block session that the Rump FS
server takes as input:

~~~
...
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
...
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

So, lets make *avconv* happy by creating a specific *mmap* implementation. In
this specific use case were *mmap* can map the Dataspace to an arbitrary
address without special requirements it should be fine. The default
implementation of the Libc VFS *mmap* is located in `libc / vfs_plugin.cc`. As
I wanted to re-use some stuff of the default implementation and also some Libc
stuff, I wanted to stay in `libc / vfs_plugin.cc` with my new code. Thus, I
modified the default *mmap* so the concrete file system is merely asked for
which flavour to use and in case of the "direct mapping" flavor also for the
targeted Dataspace.
