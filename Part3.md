---
layout: page
title: Plugin Development - Part 3
sidebar_link: true
permalink: part3/
---

![cover](https://cc3dadvancedtuts.files.wordpress.com/2015/12/interestingimage3.png)

Welcome to the final part of this tutorial series! In this section, we’ll create a CC3D project. Open Twedit++, and in the menu select CC3DProject > New CC3D Project…I like to separate Python and XML, so we’ll use that format for this section.

![Picture1](https://cc3dadvancedtuts.files.wordpress.com/2015/12/picture1.png?w=300&h=212)

Of course, add a cell type here. I’ll call it ‘Cell,’ very originally.

![Picture2](https://cc3dadvancedtuts.files.wordpress.com/2015/12/picture2.png?w=300&h=250)

Finally, I we’ll use the following plugins. To follow along, add them to your project.

![Picture3.png](https://cc3dadvancedtuts.files.wordpress.com/2015/12/picture3.png?w=736)

Alright, now you’re file structure should look like this:

- CirculizerProject/
  - CirculizerProject.cc3d
  - Simulation/
    - CirculizerProject.py
    - CirculizerProject.xml
    - CirculizerProjectSteppables.py

First, we need to activate our plugin and make sure it loaded correctly. To do so, go to the CirculizerProject.xml file and add the following:

````xml
<Plugin Name="Circularizer"></Plugin>
````

Now, open CirculizerProject.cc3d in CC3D and make sure the plugin shows up in the model editor panel.

![Picture4](https://cc3dadvancedtuts.files.wordpress.com/2015/12/picture4.png?w=300&h=227)

If it’s there, let’s change the energy penalty like we set up to do:

````html
<Plugin Name=Circularizer">
    <Penalty>999</Penalty>
</Plugin>
````

Now, we need to make sure that the methods we declared in c++ are accessible in python. How? Remember how everything we did in c++ was contained within a c++ class? We need to get ahold of the bridge to that class. Fortunately, the python object ‘CompuCell’ keeps track of this for us. So, in CirculizerProject.py, add the following:

````python
import CompuCellExtraModules
myPlugin=CompuCellExtraModules.getCirculizerPlugin()
````

Now that we have access to the c++ class, we need to pass this reference to the steppables file, so that we can use it to manipulate cells. In CirculizerProject.py, change the GrowthSteppable Instance declaration to:

````python
GrowthSteppableInstance=GrowthSteppable(_simulator = sim,_frequency=1, _cplugin = myPlugin)
````

And, in the CirculizerProjectSteppables.py file, change the
GrowthSteppable class init function  to:

````python
def __init__(self,_simulator,_frequency, _cplugin):
    SteppableBasePy.__init__(self,_simulator,_frequency)
    self.cplugin = _ plugin
````

Finally, to make sure the plugin plugged in correctly, add the following to the GrowthSteppable class:

````python
def start(self):
    import inspect
    for memberfunction in inspect.getmembers(self.cplugin, predicate=inspect.ismethod):
    print(memberfunction[0])
````

This will print all the functions in our plugin. Make sure that ‘setCellRadius’ appears within the console output.

![Picture5](https://cc3dadvancedtuts.files.wordpress.com/2015/12/picture51.png?w=257&h=300)

If it doesn’t, close CompuCell, open terminal and execute make clean; make install. Then, reopen the project.

Before we go any further, lets do some housekeeping. In the ConstraintInitializerSteppable class, change `cell.targetVolume=25` to `cell.targetVolume = cell.volume`. On the top of that file, add `import math`. And finally, in the XML file, change the `Gap` element in the UniformInitializer element to 5.

Okay, now we do the thing! In the GrowthSteppable class, let’s change the step function to:

````python
def step(self,mcs):
    for cell in self.cellList:
    self.cplugin.setCellRadius(cell, math.sqrt(cell.volume/3.141))
````

By the way, r = √(v / ∏) is just a restatement of the familiar v=∏r², which relates the volume and radius of a circle. Thus, the cell will happily become a circle with a radius that corresponds to its current volume.

And that’s it! I suggest here you tweak the project files to study the plugin. Perhaps make it animated, see the effect of changing the penalty, or combine it with other CompuCell functions.

Here is a [link](https://www.dropbox.com/s/91f0n998f9n6ftf/Circulizer.zip?dl=0) to the project files, so you can double-check your code.

Appendix: How do you access your plugin when its in the Main Code?

Simply replace

````python
import CompuCellExtraModules
myPlugin=CompuCellExtraModules.getCirculizerPlugin()
````

with

````python
import CompuCell
myPlugin = CompuCell.getCirculizerPlugin()
````
