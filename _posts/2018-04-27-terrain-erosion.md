---
author: Axel
title: Terrain Erosion on the GPU
featimg: 1.jpg
tags: [GPU, Terrain, Procedural, Erosion]
category: [standard]
---
I have been playing with different type of terrain erosion lately and one thing I would like to do is implementing all the things
I do on GPU. Erosion algorithms take many iterations to converge and are very costly when done on CPU. Most of these algorithms take advantage of parallelism: many have
been implemented on GPU, but there is not always an open source implementation. 
This is the first article of a series about terrain erosion and procedural generation. I will try to implement the things I find the most interesting, both on CPU and GPU to compare results 
(and also because compute shaders are fun). Let's start by taking a look at the state of the art on terrain erosion.
 
###### State of the Art

There are different type of erosion:
* Thermal Erosion: this is defined as "the erosion of ice-bearing permafrost by the combined thermal and mechanical action of moving water". It is the simplest one to implement but does not give realistic results by itself.
* Hydraulic Erosion: simulates water flows over the terrain. There are different types of Hydraulic erosion, but all are tricky to implement. Combined with Thermal erosion, it can give realistic looking terrain.
* Fluvial Erosion: it's the erosion of the bedrock material and its transportation downhill by streams. Usually modeled by the stream power equation as denoted by Cordonnier in 2016.

Musgrave was the first to show some results on both Thermal and Hydraulic erosion. These algorithms were ported to the GPU by Št’ava in 2008 and Jako in 2011. You can also find a very good implementation of Hydraulic Erosion
in Unity by [Digital-Dust](https://www.digital-dust.com/single-post/2017/03/20/Interactive-erosion-in-Unity).

###### Thermal Erosion

Thermal erosion is based on the repose or talus angle of the material. The idea is to transport a certain
amount of material in the steepest direction if the talus angle is above the threshold defined the material.

This process leads to terrains with a maximum slope that will be obtained by moving matter downhill. By chance, the algorithm is easily portable to the GPU: 
in fact, the core algorithm is almost identical to the CPU version. The difficulty resides in which buffer we use, how many we use and how much we care about race condition.

###### The race condition

GPU are parallel by nature: hundreds of threads are working at the same time. Thermal erosion needs to move matter from a grid point to another and we can't know which one in advance. 
Therefore, multiple threads can be adding or removing height on the same grid point. This is called a race condition and it needs to be solved in most cases.

Sometimes however we are lucky: after trying a few version of the algorithm, I found that the best solution was to just not care about the race condition happening.

###### The solution(s)

There are multiple ways to solve this problem. My first implementation used a single integer buffer to represent height data. I had to use integers because the atomicAdd function doesn't exist for floating point values. 
This solution worked and was faster than the CPU version but could only handle erosion on large scale (amplitude > 1 meter) because of integers.


In my next attempt I used two buffers: a floating value buffer to represent our height field data, and an integer buffer to allow the use of the [atomicAdd](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/atomicAdd.xhtml) glsl function. 
The floating point values were handled with [intBitsToFloat](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/intBitsToFloat.xhtml) and [floatBitsToInt](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/floatBitsToInt.xhtml) functions. 
You also have to use a barrier to make sure your return buffer is filled properly with the correct final height. This solution worked as intended and was also faster than the CPU version but slower than my previous implementation because of the two buffers. 
The main advantage of this method is that we are no longer limited by the use of integers.


My last idea was the one that I should have tried in the first place: simply ignore the race condition and use a single floating point value buffer to represent height data. Of course, the result will not be deterministic and 
will contain errors but at the end, the algorithm will converge to the same results after a few hundreds more iterations. Another good thing with this version is that we don't have any visually disturbing errors. 
The results are very similar to the other methods and this is the fastest, simplest method for now.

Here is a code snippet of the last method:

```cpp
layout(binding = 0, std430) coherent buffer HeightfieldDataFloat
{
    float floatingHeightBuffer[];
};

uniform int gridSize;
uniform float amplitude;
uniform float cellSize;
uniform float tanThresholdAngle;

bool Inside(int i, int j)
{
    if (i < 0 || i >= gridSize || j < 0 || j >= gridSize)
        return false;
    return true;
}

int ToIndex1D(int i, int j)
{
    return i * gridSize + j;
}

layout(local_size_x = 1024) in;
void main()
{
    uint id = gl_GlobalInvocationID.x;
    if (id >= floatingHeightBuffer.length())
        return;
	
    float maxZDiff = 0;
    int neiIndex = -1;
    int i = int(id) / gridSize;
    int j = int(id) % gridSize;
    for (int k = -1; k <= 1; k += 2)
    {
        for (int l = -1; l <= 1; l += 2)
        {
            if (Inside(i + k, j + l) == false)
                continue;
            int index = ToIndex1D(i + k, j + l);
            float h = floatingHeightBuffer[index]; 
            float z = floatingHeightBuffer[id] - h;
            if (z > maxZDiff)
            {
                maxZDiff = z;
                neiIndex = index;
            }    
        }
    }
    if (maxZDiff / cellSize > tanThresholdAngle)
    {
        floatingHeightBuffer[id] = floatingHeightBuffer[id] - amplitude;
        floatingHeightBuffer[neiIndex] = floatingHeightBuffer[neiIndex] + amplitude;
    }
}
```

You can see some results in the following figures.

<img src="https://raw.githubusercontent.com/Moon519/moon519.github.io/master/images/thermalResults.png">

<center>
<i>The base height fields on the left and the results of three hundred thermal erosion iteration on the right</i>
</center>

###### Results

I ran a quick benchmark to compare all the method I tried. Here are the results after 1000 iterations:

<img class="axelImg" src="https://raw.githubusercontent.com/Moon519/moon519.github.io/master/images/thermalbench.png" width="70%">

<center>
<i>On the left, a comparison between all the methods on small grid resolution. On the right, bigger resolution without the CPU version. All time are in seconds.
I didn't try to increase the grid resolution past 1024 on CPU because it took too much time, hence the two separate graphics</i>
</center>

<br/>

As expected, the single floating point buffer is the most efficient one: there is no conversion back and forth between integers and floats, and only one buffer to handle. This is an interesting solution because we compensate our 
error by increasing iteration count, which is not the most elegant but the most efficient way according to my benchmark in this case.

Code is available here: [C++](https://github.com/vincentriche/Outerrain/blob/master/Outerrain/Source/gpuheightfield.cpp) and [glsl](https://github.com/vincentriche/Outerrain/blob/master/Shaders/HeightfieldThermalWeathering.glsl).

###### References

[Interactive Erosion in Unity - Digital Dust](https://www.digital-dust.com/single-post/2017/03/20/Interactive-erosion-in-Unity)

[Interactive Terrain Modeling Using Hydraulic Erosion - Ondrej Št’ava](http://hpcg.purdue.edu/bbenes/papers/Stava08SCA.pdf)

[Fast Hydraulic and Thermal Erosion on the GPU - Balazs Jako](http://old.cescg.org/CESCG-2011/papers/TUBudapest-Jako-Balazs.pdf)

[Large Scale Terrain Generation from Tectonic Uplift and Fluvial Erosion - Guillaume Cordonnier et al.](https://hal.inria.fr/hal-01262376/document)

[The Synthesis and Rendering of Eroded Fractal Terrains - Kenton Musgrave et al.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.27.8939&rep=rep1&type=pdf)