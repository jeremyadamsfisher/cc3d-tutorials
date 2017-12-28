---
layout: page
title: Plugin Development - Part 1
sidebar_link: true
permalink: part1/
---

### Introduction

Like the [previous tutorial](https://cc3dadvancedtuts.wordpress.com/2015/08/01/manipulating-the-energy-term-for-a-pixel-flip-attempt-in-python/), we will be manipulating the energy term for a pixel flip attempt to make the cells in the simulation circular. However, we will do this in c++, rather than python because

- It is more efficient by far; and
- Accessing cell parameters does not cause a crash, as it would if we were to use Python.

However, this means that we will have to compile everything manually, and this will make our code more difficult to share. Therefor, this is a kind of last resort, though it is a powerful one.

In Part 1, we will set up the development environment. We’ll download and install the CompuCell source code, add template files for our code and generate the makefiles. In Part 2, we will write the C++ code including the python wrapper. Finally, in Part 3, we will add this code to a project, and learn how to change the parameters within python.

### Git it
You will need the source code to develop plugins. If you haven’t already, make a folder like ‘CC3D_GIT’ in your home folder and execute the following command:
`cd ~/CC3D_GIT; git clone https://github.com/CompuCell3D/CompuCell3D.git .`

### Generating the template files
Open up twedit, and click CC3D C++ and Generate New Module. ![screen](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen.jpg)

![Untitled](https://cc3dadvancedtuts.files.wordpress.com/2015/09/untitled.png?w=300&h=257)

We need to change a few things for our purpose. Just follow the screenshot: First, give it a name like ‘Circulizer.’ Then, make the code layout ‘Developer Zone.’ Make it a plugin, not a steppable. Check ‘Python Wrapper,’ ‘Attach cell variable’ and ‘ Check ‘EnergyFunction.’ Uncheck ‘LatticeMoniter’ and ‘Stepper,’ if they are enabled.

Now, the module directory should point to the ‘DeveloperZone’ folder.This will be within the CompuCell3D folder, which is in the root directory of your git cloned source code.

Click okay.

### Setting up the development environment

This part of the tutorial comes from the CompuCell developers’ youtube tutorial. If you would prefer to follow this in video form, click [here](http://www.youtube.com/watch?v=0Y2d54S3dcw&t=5m30s). I’ve linked the video so that it will start at the relevant portion.

Anyways, before we change anything in the source code, we’ll need to set up an environment. This means you will need to install all the dependancies required to compile the CC3D, and I will assume you have done so already. If not, do so for your operating system as per the instructions [here](http://www.compucell3d.org/SrcBin). (Look for the links under ‘Developers’ Corner: Building CompuCell3D From Source.’) You will also need to install CompuCell, ideally in a directory like ~/CC3D_INSTALL.

Alright. Now, launch cmake, either by opening the gui or typing `cmake-gui` into the terminal. To download the gui frontent, visit their website [here](https://cmake.org/download/).

The first thing is to fill out these fields:

[![Screen2](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen21.png?w=736)](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen21.png)

The first field should point to your the DeveloperZone folder into which you generated the plugin files earlier. On my drive, the location is ~/CC3D_GIT/CompuCell3D/DeveloperZone. The other field can be anywhere, just remember where you want to put it. I have it in my home folder under the name “DevZoneBuild.” Then, click ‘Configure.’

[![Screen3](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen3.png?w=736)](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen3.png)

You’ll get this dialog, but the default option should be fine.

[![Screen4](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen4.png?w=300&h=232)](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen4.png)

Click finish.

Now, you have to configure some of these settings. From top to bottom (more or less):

- Change the build type to ‘Release’.
- Change CMAKE_INSTALL PREFIX ### and ### COMPCUCELL3D_INSTALL_PATH  to the directory into which you installed CompuCell. For me, this is ~/CC3D_INSTALL.
- Change COMPCUCELL3D_FULL_SOURCE_PATH to ~/CC3D_GIT/CompuCell3D/, or whatever equivalent folder that may be on your hard drive.

Click Configure and then Generate. Now, close cmake. (By the way, make sure not to move any of the aforementioned folders around. If you do, you will need to repeat this whole process to re-generate the files for the next section.)

### Make it
Now we are going to make sure everything works. Open terminal and type `cd` and the directory you specified in the “Where to build the binaries” field in make. Press return, then type `make` and press return again.

It should look something like this:

[![Screen5](https://cc3dadvancedtuts.files.wordpress.com/2015/09/screen5.png?w=300&h=152)](https://cc3dadvancedtuts.files.wordpress.com/2015/09/screen5.png)

If you have an error in your code, this is where the compiler will tell you about it. But, since we have only added template files, there shouldn’t be any issue here.

If this all goes smoothly, let it finish and then type `make install`. *This* is the process where your code is actually integrated into the CompuCell application.

We’ll leave it there for now. In parts 2 and 3, we will write the code in c++ and learn how to control our plugin from python and XML. Stay tuned!
