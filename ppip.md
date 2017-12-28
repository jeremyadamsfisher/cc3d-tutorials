---
layout: page
title: Pixel Flips in Python
sidebar_link: true
permalink: ppip/
---

![cover](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen4.jpg)

### Introduction

In this tutorial, we will be manipulating the energy term for a pixel flip attempt to make the cells in the simulation circular. For now, we’ll do this in python.

In general, there are a few disadvantages to changing the energy term in python, including

- It slows down the simulation considerably;
- It is poorly documented and often crashes (without error messages!); and
- Attempting to access a cell parameter, including its dictionary or type, will always result in a crash.

However, it is also an excellent way to quickly prototype a plugin before implementing it. For simple simulations, I prefer it. The simulation itself runs more slowly, but you’ll be able to code it far more quickly.

For this tutorial, I will assume you have a modest knowledge of python and the how CompuCell works. If you unsure of what an energy term is, look at another tutorial.

### Coding the Main Python File

I’ll also assume you have already created CompuCell project. Remember, you register your steppables with the Steppable registry, as such:

````Python
steppableRegistry=CompuCellSetup.getSteppableRegistry()

from ProjectSteppables import mySteppable
steppableInstance=mySteppable(sim,_frequency=1)
steppableRegistry.registerSteppable(steppableInstance)
````

Energy terms are organized by the ‘Energy Registry,’ which is analogous to the Steppables Registry. We’ll get the ‘Energy Registry’ and register our Energy Term Function in a similar way:

````python
energyFunctionRegistry=CompuCellSetup.getEnergyFunctionRegistry(sim)

from ProjectSteppables import myEnergyFunction
myEnegyFunction=ElongateEnergyFunction(energyFunctionRegistry)
energyFunctionRegistry.registerPyEnergyFunction(myEnegyFunction)
````

That’s it for the main file. It should look like this:

[![Screen1](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen1.jpg?w=736&h=489)](https://cc3dadvancedtuts.files.wordpress.com/2015/08/screen1.jpg)

### Coding the Steppables Python File

We’ll actually code the energy function within the ‘Steppables’ file, like a steppable (even though it isn’t). First, we’ll need to import a class to make this all work:

````python
from PyPlugins import *
````

We’ll also use the math library in this tutorial:

````python
import math
````

Great. Now, here is the template code you’ll need:

````python
class myEnergyFunction(EnergyFunctionPy):

    def __init__(self,_energyWrapper):
        EnergyFunctionPy.__init__(self)
        self.energyWrapper=_energyWrapper

    def changeEnergy(self):
        energy = 0

        newCell=self.energyWrapper.getNewCell()
        oldCell=self.energyWrapper.getOldCell()
        myChangePoint = self.energyWrapper.getChangePoint()

        if newCell:
            pass
        if oldCell:
            pass

        return energy
````

There are a few things to note about this. First, notice that we no longer inherit from SteppableBasePy. Instead, we’ll inherit from `EnergyFunctionPy`. Therefore, we’ll be overriding the `changeEnergy(self)` function, rather than the familiar `start(self)` or `step(self, mcs)`functions. This is where the interesting stuff happens!

Second, pay special attention to these lines:

````python
newCell=self.energyWrapper.getNewCell()
oldCell=self.energyWrapper.getOldCell()
myChangePoint = self.energyWrapper.getChangePoint()
````

At this point, understanding what these represent is essential to properly change the Energy Term. Recall, during a Pixel Flip Attempt, CC3D randomly chooses two adjacent points. One of these is the ‘change point,’ which belongs to the `oldCell`, and the other is the ‘flip point,’ which belongs to the `newCell`. Then, CC3D asks “of these two cells, which would __prefer__ to have the pixel at the change point?” Stated more rigorously, it calculates whether it would be energetically favorable to change the identity (or ownership) of the pixel from the `oldCell`to the `newCell`. (This will be the case when the sum of __all__ the energy terms, from __all__ the different plugins, turns out negative.) This is the context in which your code here will be evaluated.

Take note: often, CC3D picks a pixel for a Pixel Flip Attempt that does not belong to any particular cell; rather, it belong to the medium. In this case, the function `self.energyWrapper.getNewCell()` returns a `NULL` pointer. You can check if this is the case with the following code:

````python
if(cell): #if this is NOT medium, then...
    self.foo(bar)
````

Without this code, CC3D will eventually try to access a variable, like `cell.type`, from a `NULL`pointer. This will cause it to crash.

At this point, you know have everything you need to change the energy term of a Pixel Flip Attempt. Go wild! I will just share this example code that imposes a penalty on pixel flip attempts outside of a certain radius.

````python
class myEnergyFunction(EnergyFunctionPy):

    def __init__(self,_energyWrapper):
        EnergyFunctionPy.__init__(self)
        self.energyWrapper=_energyWrapper

    def calculateDistance(self, _from, _to):
        distance_squared = 0.0
        for i in xrange(len(_from)):
            distance_squared += (_from[i] - _to[i])**2
        return math.sqrt(distance_squared)

    def changeEnergy(self):
        energy = 0
        energy_penalty = 99999

        newCell=self.energyWrapper.getNewCell()
        oldCell=self.energyWrapper.getOldCell()
        myChangePoint = self.energyWrapper.getChangePoint()

        target_radius = 11.17

        if newCell:
            distance = self.calculateDistance(_to = [newCell.xCOM,newCell.yCOM], _from = [myChangePoint.x, myChangePoint.y])
            if distance &gt; target_radius:
                energy += energy_penalty
        if oldCell:
            distance = self.calculateDistance(_to = [oldCell.xCOM,oldCell.yCOM], _from = [myChangePoint.x, myChangePoint.y])
            if distance &gt; target_radius:
                energy -= energy_penalty

        return energy
````

Lets break this down.

Now, `calculateDistance` is your straightforward distance calculating function. `__init__` doesn’t really do anything here.

The action happens in the `changeEnergy()` function. The very first thing I do is declare a variable, `energy`. At the end of this function, we’ll return this variable to represent the change in energy of changing the pixel identity from the `oldCell` to the `newCell`. Doing so allows us to consider the `newCell` and `oldCell` separately.

Now, I want each cell to be a circle with a radius of 11.17.This is the radius of a circle with a volume (i.e. surface area) of 100 pixels, which is the volume I specified in the `UniformInitializer`. So, I declare this variable and set it aside.

Now, we’ll consider each cell in turn. Notice, we’ll have to check that these cells __aren’t__ medium first, to protect against crashing. So, we’ll consider them within an `if()` statement:

````python
if(newCell): #if this is NOT medium, then...
    if(doIwantThisPixelOrNot() == True):
        energy -= 99999
    else:
        energy += 99999
````

Then, we’ll ask whether the distance between the change point and the `newCell` center of mass is greater than the target radius. If so, we’ll impose a penalty. This discourages the newCell from having any pixel outside its circular region.

Similarly, we’ll ask whether the distance between the change point and the `oldCell` center of mass is greater than the target radius. This time, we’ll *decrease* the change in energy if this is the case. Why? Because, if the pixel flip succeeds, the `oldCell` will keep the pixel at the change point. Thus, if the change point is outside its circular region, we want the pixel flip attempt to *succeed*.

And that’s about it. Well done!

The full code can be found here: <https://www.dropbox.com/s/tgl30iyl6u9ju1j/MakeCellsCircular.zip?dl=0>
