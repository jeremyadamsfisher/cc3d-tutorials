---
layout: page
title: Plugin Development - Part 2
sidebar_link: true
permalink: part2/
---

![cover](https://cc3dadvancedtuts.files.wordpress.com/2015/09/interestingimage21.png)

Welcome back! At the beginning of this tutorial, you should have set up all the needed files, as well as a method to debug your plugin. By the end, you will know how to program a Change Energy Function, access XML attributes/elements and store additional variables in each cell.

First, the file structure should look like this:

- CirculizerData/
  - CirculizerData.h
  - CirculizerDLLSpecifier.h
  - CirculizerPlugin.cpp
  - CirculizerPlugin.h
  - CirculizerPluginProxycpp
  - CMakeLists.txt

### Overview

In this tutorial, we will implement three methods. The first will control the ‘changeEnergy’ term (see: [CompuCell Theory](/theory/)) of the plugin, which represents the "guts" of our project. The other two will allow you to ‘steer’ your plugin through XML and python.

### Programming the Change Energy Function

Lets begin with the most important aspect of our project: the ‘changeEnergy’ function. We will expand upon the following skeleton code in the CircularizerPlugin.cpp file:

````c++
double CirculizerPlugin::changeEnergy(const Point3D &pt,const CellG *newCell,const CellG *oldCell) {    

    double energy = 0;
    if (oldCell){
        //PUT YOUR CODE HERE
    }else{
        //PUT YOUR CODE HERE
    }

    if(newCell){
        //PUT YOUR CODE HERE
    }else{
        //PUT YOUR CODE HERE
    }

    return energy;
}
````

Lets clarify the function parameters. The pointer `&pt` refers to the "change point," which is a randomly chosen coordinate in the simulation lattice. CompuCell will decide, based on the output of all plugins’ `changeEnergy` functions, whether to assign this pixel to a new owner (`newCell`) or to its previous owner (`oldCell`). The other parameters refer to these potential owners: `*newCell` and `*oldCell`. (Note that cells, in the Potts model, are simply a collection of pixels.)

So, lets think about this. We need to make our cells circular, and we can do that by controlling the energetics of membrane fluctuations. So, we need to program it such that membrane fluctuations that cause the membrane to expand __beyond a specified radius__ are unfavorable (and membrane fluctuations that keep the cell within this radius to be favorable or neutral).

In terms of changeEnergy, oldCell and newCell, this means that — when a cell (newCell) is thinking about adding a pixel to itself, it needs to ask: is this pixel beyond this distance certain radius? Let’s convert this to c++ code.

````c++
double radius = 10.0
if(newCell){
        double a = newCell->xCOM - pt.x;
        double b = newCell->yCOM - pt.y;
        double c = sqrt(pow(a,2) + pow(b,2));
        if(c > radius){
                energy += 999;
        }
}
````

First, this is all wrapped in a `if(newCell)` statement, to make sure we don’t reference a non-existing variable. (It would cause an error if we didn’t do this.) Then, we do the pythagorean formula to calculate the distance between the newCell’s center of mass and the change point. (`sqrt(x)` outputs the square root of x and `pow(x,2)` outputs x squared.) Finally, we ask, is this distance greater than our specified radius, which is 10.0 in this case? If so, then we impose an energy penalty of 999.

We also need logic to deal with the oldCell, to which the changePoint pixel previously belonged. This is the opposite logic: when the distance of the changePoint to the oldCell’s center of mass is greater than the specified radius, then we make the pixel flip energetically favorable.

Add this just below the above code.

````c++
if(oldCell){
        double a = oldCell->xCOM - pt.x;
        double b = oldCell->yCOM - pt.y;
        double c = sqrt(pow(a,2) + pow(b,2));

        if(c > radius){
            energy -= 999;
        }
    }
````

That should do it for now.

**Make it python steerable**

The problem with the above code is that it *only* makes cells into circles with a radius of 10. What if we wanted to control this radius from python?

This is where the "attach cell attribute" checkbox comes in. You should have a files called ‘CirculizerData.h’. Open that and add a line under `int x;`:

````c++
int circle_radius;
````

That will store an extra variable in each cell.

Now, we’ll need a function to change this variable, which will be available within python. To do so, open the CirculizerPlugin.h file and add the following function declaration under `public:`:

````c++
virtual void setCellRadius(CellG *Cell, float _radius);
````

To flesh out this function, go to the CirculiZerPlugin.cpp file and add the following anywhere:

````c++
void CirculizerPlugin::setCellRadius(CellG *Cell, float _radius){
    circulizerDataAccessor.get(Cell->extraAttribPtr)->circle_radius = _radius;
}
````

What does this mean? Well, circulizerDataAccessor is an object that accesses cells’ variables that are added by plugins. (It does not, however, access the cells’ default variables, such as its `type` or `targetVolume` variables.) Calling `circulizerDataAccessor.get(Cell->extraAttribPtr)->circle_radius` gives us the variable we declared in CirculizerData.h. We set this variable to the parameter we will provide on the python end. Of course, `Cell` also refers to the a `cell` object that we will provide in python.

So, we can declare and set this radius variable, but it doesn’t *do anything*. We need to make the `changeEnergy` function use our variable instead of the constant 10.0.

Replace the `changeEnergy` code to this:

````c++
double CirculizerPlugin::changeEnergy(const Point3D &pt,const CellG *newCell,const CellG *oldCell) {
    double energy = 0;

    if (oldCell){
        double radius = circularizerDataAccessor.get(oldCell->extraAttribPtr)->circle_radius;

        double a = oldCell->xCOM - pt.x;
        double b = oldCell->yCOM - pt.y;
        double c = sqrt(pow(a,2) + pow(b,2));

        if(c > radius){
            energy -= 999;
        }else{
            energy += 999;
        }
    }

    if(newCell){
        double radius = circularizerDataAccessor.get(newCell->extraAttribPtr)->circle_radius;

        double a = newCell->xCOM - pt.x;
        double b = newCell->yCOM - pt.y;
        double c = sqrt(pow(a,2) + pow(b,2));

        if(c > radius){
            energy += 999;
        }else{
            energy -= 999;
        }
    }

    return energy;
}
````

That should do it! We may now tell cells what their radius should be.

### Make it XML steerable

There is one final thing inelegant in our code: the 999 energy penalty. What if, in the middle of developing our model, we realize that that value is too high or too low? We don’t want to recompile the whole source tree. Rather, we want to we able to change the value within the model’s python or XML. For variety, we’ll make our plugin listen to XML.

By default, the function that parses the XML looks something like this:

````c++
void DevZoneTestPlugin::update(CC3DXMLElement *_xmlData, bool _fullInitFlag){
    //PARSE XML IN THIS FUNCTION
    //For more information on XML parser function please see CC3D code or lookup XML utils API
    automaton = potts->getAutomaton();
    ASSERT_OR_THROW("CELL TYPE PLUGIN WAS NOT PROPERLY INITIALIZED YET. MAKE SURE THIS IS THE FIRST PLUGIN THAT YOU SET", automaton)
   set<unsigned char> cellTypesSet;

    CC3DXMLElement * exampleXMLElem=_xmlData->getFirstElement("Example");
    if (exampleXMLElem){
        double param=exampleXMLElem->getDouble();
        cerr<<"param="<<param<<endl;
        if(exampleXMLElem->findAttribute("Type")){
            std::string attrib=exampleXMLElem->getAttribute("Type");
            // double attrib=exampleXMLElem->getAttributeAsDouble("Type"); //in case attribute is of type double
            cerr<<"attrib="<<attrib<<endl;
        }
    }

    //boundaryStrategy has information aobut pixel neighbors
    boundaryStrategy=BoundaryStrategy::getInstance();

}
````

This code looks for an element named "Example," takes its double value, looks within the element for a "Type" attribute, then prints both values to the console. (For the difference between an attribute and an element, see: <http://www.w3schools.com/xml/xml_dtd_el_vs_attr.asp>) It would correspond to an XML like this:

````xml
<Plugin Name="Circularizer">
    <Example type="strong_and_silent">666.0</Example>
</Plugin>
````

Let’s change this to work for us. First, we need to keep track of this variable. The best way to do so is to declare a class variable. In the CirculizerPlugin.h file, add the following under the line `public:`:

````c++
double xml_energy_penalty;
````

And change the `update` function like so:

````c++
void CirculizerPlugin::update(CC3DXMLElement *_xmlData, bool _fullInitFlag){
    automaton = potts->getAutomaton();
    ASSERT_OR_THROW("CELL TYPE PLUGIN WAS NOT PROPERLY INITIALIZED YET. MAKE SURE THIS IS THE FIRST PLUGIN THAT YOU SET", automaton)
    set<unsigned char> cellTypesSet;

    CC3DXMLElement * myElement = xmlData->getFirstElement("Penalty");
    if(myElement){
        xml_energy_penalty = myElement->getDouble();
    }else{
        xml_energy_penalty = 99999;
    }

    //boundaryStrategy has information aobut pixel neighbors
    boundaryStrategy=BoundaryStrategy::getInstance();
}
````

This code simply looks at the first element called "Penalty" and takes its value. If this value is not specified, then it defaults to 999.0. This would correspond to an XML file like such:

````xml
<Plugin Name="Circulizer>
    <Penalty>666.0</Penalty>
</Plugin>
````

Finally, lets take this `xml_energy_penalty` variable and factor it into the all-important `changeEnergy` function! To do so, simply replace the 999’s with `xml_energy_penalty`, like so:

````c++
double CirculizerPlugin::changeEnergy(const Point3D &pt,const CellG *newCell,const CellG *oldCell) {
    double energy = 0;

    if (oldCell){
        double radius = circularizerDataAccessor.get(oldCell->extraAttribPtr)->circle_radius;

        double a = oldCell->xCOM - pt.x;
        double b = oldCell->yCOM - pt.y;
        double c = sqrt(pow(a,2) + pow(b,2));

        if(c > radius){
            energy -= xml_energy_penalty;
        }else{
            energy += xml_energy_penalty;
        }
    }

    if(newCell){
        double radius = circularizerDataAccessor.get(newCell->extraAttribPtr)->circle_radius;

        double a = newCell->xCOM - pt.x;
        double b = newCell->yCOM - pt.y;
        double c = sqrt(pow(a,2) + pow(b,2));

        if(c > radius){
            energy += xml_energy_penalty;
        }else{
            energy -= xml_energy_penalty;
        }
    }
    return energy;
}
````

That’s it! Now, all that remains to do with the source code is to compile it. At this point, since we have declared so many new things, it is good practice to clean all the cached files. In terminal, `cd` into the "CC3D_Build" directory and execute `make clean`. Then, execute `make install`.

This is a good point to make a cup of tea. (Matcha green is my favorite flavor.) It will take some time to recompile from scratch.

![powderedgreentea](https://cc3dadvancedtuts.files.wordpress.com/2015/09/powderedgreentea.jpg?w=300&h=200)

(Photo credit: [Miketsukunibito~commonswiki](https://commons.wikimedia.org/wiki/User:Miketsukunibito~commonswiki))

Next time, we will access and control the plugin within a CompuCell project. Stay tuned!

Attached in this [link](https://www.dropbox.com/s/91f0n998f9n6ftf/Circulizer.zip?dl=0) are the plugin files, so you can double check your code.
