GPU Hardware For Signed Distance Fields

The Signed Distance Field (SDF) is a fundamental primitive used in computer graphics, which has the nice property that it encodes a surface in 3D space, which regardless of topological complexity can be filtered efficiently for automatic level-of-detail by massively parallel hardware, much as a texture map can.

This makes SDF amenable to implementation in GPU hardware, though to date no hardware has been designed specifically to accelerate SDF. Rather, whatever hardware was developed for triangle rasterization has been repurposed for SDF by end users.

In this article we explore possible changes to GPU hardware specifically for the purpose of accelerating SDF.

Texture Format

For any given mip map of an SDF texture, the delta in value between adjacent texels has a very limited range across the entire image, which at the scale of one texel per distance unit is [0..1], and is about twice what it was in the next-bigger mip map. As this limited range is constant across the image,
it is a waste of bits to also independently encode it into each texture block. However, existing texture formats must all do this. Those bits could instead be devoted to encoding which value between [0..N] the texel is.

Likewise, if mip-map filtering is assumed then any quad of adjacent texels in which all texels are valued <=-1 or >=1, can be encoded as the constant value -1 or 1 respectively, as their values never contribute to the calculation of where the surface is in 3D space.
In practical terms this could be implemented in hardware as a "meta map" in which each entry represents NxN texels, and indicates whether all NxN are <=-1, all are >=1, or otherwise. This is 2 bits per NxN texels, which is an order of magnitude less data than
existing texture hardware formats.

Face Centered Cubic Volume Textures

![Primitive Cubic vs. Face Centered Cubic](https://wisc.pb.unizin.org/app/uploads/sites/293/2019/07/CNX_Chem_10_06_CubUntCll.png)

The Nintendo 64 game console did not tap the 4 nearest samples in a 2D grid to *bilinearly* filter a 2D texture, as contemporary hardware does. Rather, it tapped the 3 nearest samples, and *linearly* blended among them.
For a given dimension N, a bilinear filter requires pow(2,N) taps, and a linear filter requires N+1 taps. In ten dimensions bilinear would require 1024 taps, and linear 11. So, any hardware that aims to sample a high 
dimensional function would be under great pressure to use linear filtering, rather than bilinear filtering.

An N64 texture can be conceptualized as being made of tiny triangular cells, rather than the square cells of contemporary hardware texture. In 3D such a texture would be made of rhombic dodecahedra, in a Face Centered
Cubic lattice (see above image) rather than the Primitive Cubic lattice of contemporary hardware.

![N64 texture sample grid](https://www.theedkins.co.uk/jo/tess/triangle10.gif)
![Contemporary texture sample grid](https://mammothmemory.net/images/user/base/Maths/Geometry/Tessellation/a-square-is-a-shape-that-can-be-tessellated.401ad26.jpg  =100x20)

An SDF encoded as a 3D texture would require 4 taps with a linear filter, rather than 8 taps with a bilinear filter. The pipeline to blend 1/2 as many taps would require 1 less bit of internal precision, and would 
process 1/2 as much data at any instant. This may present opportunities to increase the degree of parallelism by a factor of 2.
