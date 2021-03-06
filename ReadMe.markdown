About ECHelper
==============

This sample illustrates how to install a helper tool that needs to run with privileges as a launchd task, how to have launchd launch the task on demand, and how to communicate with it using IPC.

It is based on a mish-mashg of Apple sample code, including SMJobBless, SampleD and CFLocalServer.

The original samples are available from:

- http://developer.apple.com/library/mac/#samplecode/SMJobBless/Introduction/Intro.html
- http://developer.apple.com/library/mac/#samplecode/SampleD/Introduction/Intro.html
- http://developer.apple.com/library/mac/#samplecode/CFLocalServer/Introduction/Intro.html


Staying Consistent
------------------

The biggest problem with the installation part of the task is that the helper tool and the host application that's going to install it have to be set up very carefully to have the right code-signing details and bundle ids.

If you wanted to change these in the SMJobBless example you had to do it in lots of different places - and it was easy to miss one.

This sample gets round that problem by setting three user-defined values at the project level, in the Settings.xcconfig file:


    HELPER_ID = com.elegantchaos.helper.helper
    HOST_ID = com.elegantchaos.helper.host
    HELPER_SIGNING = 3rd Party Mac Developer Application: Sam Deane


You should set HELPER_ID to the bundle id that you want to use for your helper application.

You should set HOST_ID to the bundle id that you want to use for the host application - the one that installs the helper (and which probably makes use of it, although that's not necessarily the case).

You should set HELPER_SIGNING to the code signing profile that you want to use to sign everything. Note that if this profile is associated with a bundle id pattern (eg. com.elegantchaos.*) then the HELPER_ID and HOST_ID settings must match the pattern, otherwise xcode will refuse to sign the applications. 

The name of the profile is embedded in various places, and it's important that they all match exactly what's in the certificate. For this reason you need to specify an exact profile name here (like "3rd Party Mac Developer Application: Sam Deane"), rather than a wildcard (like "3rd Party Mac Developer Application: *").


Building The Plists
-------------------

The tricky part of all of this is that the host application needs one plist, and the helper application two, and both of them make reference to the three values defined above.

What's more, you can't rely on Xcode's variable substitution to fill in all of these values. Two of the plists are embedded in the __TEXT section of the helper application, which is a simple binary and therefore doesn't live in a bundle. These are embedded with some special linker commands, and it appears that Xcode (and the underlying linker tools) don't pre-process the lists in passing.

Even with the other plist (which is used for the host application) has a complication to it, because one of the special keys in it is a dictionary where a key needs to be replaced by the value of HELPER_ID. It appears that Xcode's plist pre-processing doesn't affect keys, only values, so you can't use a variable in the key to fill in the correct value.

The way I've got round this is to add build scripts to both targets which take the original plists and perform the substitutions. The various linking stages then use the output of these scripts as their inputs.

In theory these derived plists should probably go into ${DERIVED_FILE_DIR}, so that they are hidden out of the way. 

However, it seems that Xcode needs to be able to find them when the build starts, because they are set as the Info.plist to use for the targets. When I tried hiding them away in the build folder, Xcode couldn't seem to cope, so instead I've added them to the project as "proper" resource files. Each time you build, they'll be regenerated. Luckily though, Xcode (and git) don't see them as having changed unless they actually do change (because you change the values of the variables mentioned above).


IPC
---

I've also expanded the example to illustrate a couple of ways to actually communicate with the helper application.

The example uses distributed objects, which is *NOT* a particularly secure way to do things since it's quite high level and exposes an actual object interface. 

Assuming that your helper is running with enhanced priveleges of some sort, it obviously becomes a potential attack vector for things that are trying to take over your system. Distributed objects is big and powerful, and complicated, and thus hard to secure.

In reality you should probably use something simpler, like sending simple bytecode across a NSMachPort or NSSocketPort.

In my sample I've illustrated both using Mach ports and using Unix domain sockets. The unix domain sockets approach is apparently recommends, but it was *very* hard to track down enough information to get it to work cleanly, so I hope that this sample gives a few people pointers in the right direction in future...

The IPC method that you choose is determined by a fourth setting:

    HELPER_METHOD = Mach
    
or

    HELPER_METHOD = Sockets
    
This changes the launchd plist that gets used, and also filters through into the code to change the IPC method at runtime.

    
Scripts
-------

There are some handy shell scripts included, to start, load, unload and cleanup the helper, and dump out what's going on.

In particular, you should run the cleanup script when you change something. It will remove all trace of the previous helper, ensuring that you test in a clean environment. 
This script runs some sudo rm commands, so be careful what you do with it - if you manage to *really* mess things up and give it the wrong path info it could eat your hard drive!

