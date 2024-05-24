GPU Hardware For Signed Distance Fields

The Signed Distance Field (SDF) is a fundamental primitive used in computer graphics, which has the nice property that it encodes a surface in 3D space, which regardless of topological complexity can be filtered efficiently for automatic level-of-detail by massively parallel hardware, much as a texture map can.

This makes SDF amenable to implementation in GPU hardware, though to date no hardware has been designed specifically to accelerate SDF. Rather, whatever hardware was developed for triangle rasterization has been repurposed for SDF by end users.

In this article we explore possible changes to GPU hardware specifically for the purpose of accelerating SDF.

Texture Format

For any given mip map of an SDF texture, the delta in value between adjacent texels has a very limited range across the entire image, which at the scale of one texel per distance unit is [0..1], and is about twice what it was in the next-bigger mip map. As this limited range is constant across the image,
it is a waste of bits to also independently encode it into each texture block. However, existing texture formats must all do this.

If mip-map filtering is assumed then any block of NxNxN texels in which all texels are either inside or outside the SDF surface, need not encode any information, other than the fact that all texels are inside or outside the surface. For example for a 3D SDF texture we can store a much smaller "meta texture"
in which each block of 8x8x8 texels is encoded as two bits (00: all inside, 01: all outside, 02: mixed). That is 0.004 bits per texel, or 99.6% compression. If all of our filter taps are determined to be all-inside or all-outside, then the result of our texture fetch is either "inside" or "outside" with no need 
to read individual texels.

When it is necessary to read individual texels, because the 2-bit 8x8x8 cube fetch returns "mixed" results, the individual texels do not benefit from hardware texture block compression: there is no need to store block "endpoints" because each 8x8x8 block will encode the same range of values. Since for most 
texels the value can be predicted from adjacent texels (most texels are "one unit" further away from the surface than their neighbors) a simple predictive decoder in hardware is likely far more efficient. 

Face Centered Cubic Volume Textures

![Primitive Cubic vs. Face Centered Cubic](https://wisc.pb.unizin.org/app/uploads/sites/293/2019/07/CNX_Chem_10_06_CubUntCll.png)

The Nintendo 64 game console did not tap the 4 nearest samples in a 2D grid to *bilinearly* filter a 2D texture, as contemporary hardware does. Rather, it tapped the 3 nearest samples, and *linearly* blended among them.
For a given dimension N, a bilinear filter requires pow(2,N) taps, and a linear filter requires N+1 taps. In ten dimensions bilinear would require 1024 taps, and linear 11. So, any hardware that aims to sample a high 
dimensional function would be under great pressure to use linear filtering, rather than bilinear filtering.

An N64 texture can be conceptualized as being made of tiny triangular cells, rather than the square cells of contemporary hardware texture. 

![N64 texture sample grid](https://www.theedkins.co.uk/jo/tess/triangle10.gif)

In 3D such a texture would be made of rhombic dodecahedra, in a Face Centered Cubic lattice (see above image) rather than the Primitive Cubic lattice of contemporary hardware.

![Rhombic Dodecahedron Honeycomb](https://upload.wikimedia.org/wikipedia/commons/2/2e/Rhombic_dodecahedral_honeycomb_4-color.gif)

An SDF encoded as a 3D texture would require 4 taps with a linear filter, rather than 8 taps with a bilinear filter. The pipeline to blend 1/2 as many taps would require 1 less bit of internal precision, and would 
process 1/2 as much data at any instant. This may present opportunities to increase the degree of parallelism by a factor of 2.
