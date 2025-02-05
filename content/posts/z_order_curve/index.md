---
title: "Memory Mapping Point Clouds using Z-order curves"
date: 2025-02-03T22:21:50+05:30
draft: false
tags: [point-clouds,optimization,interactive]
categories: [mathematics,3D,programming]
---

## The Problem
While prototyping different algorithms that work on point clouds, I would often run into an issue. although I had point clouds with just a few million points for prototyping, when testing the algorithms on larger clouds, with 100s of millions or billions of points, the applications quickly became memory intensive taking most of available memory (10s of gigabytes).

oftentimes the point clouds also come with additional data attached to each point making them resource intensive to work with. so I Looked for ways to optimize ways of loading, querying and storing the point clouds for easier processing.


the first thing I did was to separate the xyz coordinate data from the point cloud. one operation I cared the most about and most people would often find themselves doing is getting a cropped section of the pointcloud, like a bounding box crop. 

ideally we create data structures such as a KD-Tree or an Octree from a pointcloud, that can crop the pointcloud in logarithmic time. but since the whole cloud would have to be loaded in memory, any method that uses in-memory optimization will be resource intensive.

## Two Approaches
one thing that became immediately obvious was, to create an Octree or KD-Tree once, break the pointcloud into grid cells. write the grid cells onto disk and use a map to load points dynamically from the disk. we could do this in two ways, either we write points in each grid cell to its own file.

> **_NOTE:_** All the illustrations are in 2D for convenience purposes, the concepts extend naturally to 3D

<img src="images/points_multi_file.png" alt="drawing" style="width:800px;"/>
which would mean we'd have to load a section of point cloud from multiple files, or we could merge the points from all grid cells into a single file and have an index map for querying. 

<img src="images/points_single_file.png" alt="drawing" style="width:800px;"/>

## Spatial Locality
there's one more consideration here. file-IO works faster when the data to be read is contiguous. if we write the point cloud to disk as multiple files, then the reads would be relatively expensive than reading an index range from a single file. is there a way to sort the points such that range queries would on average be contiguous ? can we sort the points or grid cells in such a way that points closer in 3D (or 2D in case of these illustrations ) are also closer when stored as a single array on disk ? can we preserve this property called **Spatial Locality** while saving the points to disk ? Space filling curves do just that. and this is how the idea of a Z-order curve or morton order curve, helped me efficiently process point clouds by improving file-IO and memory requirements of my algorithms.

## Z order - Morton order
the name Z-order comes from the zig-zag Z like pattern that emerges when we combine the bits of coordinates by interleaving them and using the resulting value to sort the points.

take four numbers for example. 0,1,2,3
in binary representation with two bits, they are 00, 01, 10 and 11
let's say you want to calculate the morton code for a point in 2D \\((2,3)\\).

## Calculating Morton Codes
binary representation of 2 is 10
binary representation of 3 is 11
to calculate the morton code, we dilate the bits of 2, in other words we make space for the bits of 3 and also dilate the bits of 3.
$$
2 = {(\color{lime}{1 \text\_ 0 \text\_})}_b
$$
$$
3 = {(\color{violet}{\text\_ 1 \text\_ 1 })}_b
$$

then we combine them both

$$
MortonCode(2,3) = \color{lime}{1}\color{violet}{1}\color{lime}{0}\color{violet}{1}
$$

similarly if we tabulate the morton codes for all 16 possible points (x,y) in 2D where x or y can be any of [0,1,2,3].
$$
\begin{array}{|c|c|c|c|c|}
\hline
y \backslash x &  \color{violet}{00} & \color{violet}{01} & \color{violet}{10} & \color{violet}{11}  \newline
\hline
\color{lime}{00} & 
\color{lime}{0}\color{violet}{0}\color{lime}{0}\color{violet}{0} & 
\color{lime}{0}\color{violet}{0}\color{lime}{0}\color{violet}{1} & 
\color{lime}{0}\color{violet}{1}\color{lime}{0}\color{violet}{0} & 
\color{lime}{0}\color{violet}{1}\color{lime}{0}\color{violet}{1} \newline
\hline
\color{lime}{01} & 
\color{lime}{0}\color{violet}{0}\color{lime}{1}\color{violet}{0} & 
\color{lime}{0}\color{violet}{0}\color{lime}{1}\color{violet}{1} & 
\color{lime}{0}\color{violet}{1}\color{lime}{1}\color{violet}{0} & 
\color{lime}{0}\color{violet}{1}\color{lime}{1}\color{violet}{1} \newline
\hline
\color{lime}{10} & 
\color{lime}{1}\color{violet}{0}\color{lime}{0}\color{violet}{0} & 
\color{lime}{1}\color{violet}{0}\color{lime}{0}\color{violet}{1} & 
\color{lime}{1}\color{violet}{1}\color{lime}{0}\color{violet}{0} & 
\color{lime}{1}\color{violet}{1}\color{lime}{0}\color{violet}{1} \newline
\hline
\color{lime}{11} & 
\color{lime}{1}\color{violet}{0}\color{lime}{1}\color{violet}{0} & 
\color{lime}{1}\color{violet}{0}\color{lime}{1}\color{violet}{1} & 
\color{lime}{1}\color{violet}{1}\color{lime}{1}\color{violet}{0} & 
\color{lime}{1}\color{violet}{1}\color{lime}{1}\color{violet}{1} \newline
\hline
\end{array}
$$

when we trace a path in ascending order of their morton codes, we see the Z-ordering.
$$
\begin{array}{|c|c|c|c|c|}
\hline
y \backslash x &  \color{violet}{0} & \color{violet}{1} & \color{violet}{2} & \color{violet}{3}  \newline
\hline
\color{lime}{0} & \color{orange}{0} & \color{orange}{1} & \color{orange}{4} & \color{orange}{5} \newline
\hline
\color{lime}{1} & \color{orange}{2} & \color{orange}{3} & \color{orange}{6} & \color{orange}{7} \newline
\hline
\color{lime}{2} & \color{orange}{8} & \color{orange}{9} & \color{orange}{12} & \color{orange}{13} \newline
\hline
\color{lime}{3} & \color{orange}{10} & \color{orange}{11} & \color{orange}{14} & \color{orange}{15} \newline
\hline
\end{array}
$$
<style>
img {
  display: block;
  margin-left: auto;
  margin-right: auto;
}
</style>
<img src="images/featured.png" alt="drawing" style="width:200px;"/>

as you can observe the points closer in 2D are also close to each other along the curve in order **most** of the time and if I wanted to read the points in the cropping box, all the numbers are contiguous and I just need one ranged query to read, from index 0 to 7.

of course it's seldom the case for real world data to be so clean. nonetheless the method can be applied to get a similar result. below is a random instance of the same, I created a set of random points in 2D in the range (0,width) and (0,height) for x and y coordinates of the points. the width and height of the canvas are close to 500 pixels, so the points are more spread apart than our earlier range [0,3] for x and y. 

by default you will only see the points. tap to cycle through different orders of points. the second mode displays the order in which points are generated. the third one displays the order after the points are sorted using their morton codes.
{{< raylibcanvas scriptPath="/posts/z_order_curve/interactive/morton.js" >}}

## contiguous ordering
when points are not sorted, all of them are randomly spread across the array at different locations or in different files if they are grouped into grid cells and written into individual files or non contiguous index ranges if the grid cells are grouped into a single file. in the illustration below, if you look at the box crop, then points that are within the box are at different indices, not all contiguous. analogously in practice this could mean that a box crop on a point cloud in 3D might result in reading multiple files or different index ranges ( not necessarily contiguous) if they are written to the same file.
<img src="images/random_arr.png" alt="drawing" style="width:500px;"/>
using morton order this gets mitigated a lot. when points are sorted using their morton codes however, you can see a pattern. not only are the points closer together in space, they are also closer together on the array **MOST** of the time, although in some cases like in the case of points 3 and 8 that are farther apart in the array even though they are closer together in 2D, this ordering is still preferrable because it performs better on average and works for any number of dimensions(3D and more), which is why I said this mitigates the problem by a great deal but we still have corner cases. so if you were to widen the cropping box horizontally, the points 4,5,6,7 will follow 0,1,2,3 and instead of two ranged queries 0-3 and 4-7 you will need just one query 0-7. this is the advantage of using morton order to sort points before storing them because doing so preserves spatial locality. the illustrations are in 2D, but the idea extends to any number of dimensions.
<img src="images/morton_arr.png" alt="drawing" style="width:500px;"/>


putting it all together, we can now sort the points in morton order. group the points from grid cells in the same order and write them to disk. we can store index ranges in a separate file. most of the time this ordering would allow for combining multiple index ranges into a single contiguous index range, reducing the time it takes to load the points queried.

<img src="images/points_morton_order.png" alt="drawing" style="width:500px;"/>



> **_NOTE:_** more specific implementation details have been skipped.
