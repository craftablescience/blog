+++
title = "Format Deep Dive: VTF (Valve Texture Format)"
date = "2024-11-26"
author = "Laura Lewis"
cover = "posts/format_deep_dive_vtf/cover.webp"
tags = ["source-engine", "file-format"]
description = "A deep dive into the evolution and inner workings of VTF files."
showFullContent = false
readingTime = true
+++

## Background / Definitions

At this point I'm going to assume you are a smart cookie and know what textures are, and why engines need to use custom
file formats to store them.

(tl;dr it's more efficient than loading more accessible formats like PNG, some types of image data cannot be stored in
accessible formats like PNG, engines frequently need to assign textures metadata which cannot be done in most accessible
formats like PNG)

From this point forward, when I say "image", "image data", or "texture data" I'm referring to a two-dimensional array of
pixels, and not a file like PNG or VTF.

Finally, I'd like to note that I have no experience with the VTF format for console versions of Source, so this post
will strictly focus on the VTF formats found on PC. Console VTFs seem to be very similar, and I might make a separate
blog post about them in the future when I have more experience working with them.

Anyway now that that's out of the way, let's get into this format.

## DDS

Since the first iteration of VTF is based on the DDS file format, let's go over that first, and then we can see what
Valve took inspiration from. Skip past this section if you already know what DDS looks like or don't particularly care.
The [DDS header specification](https://learn.microsoft.com/en-us/windows/win32/direct3ddds/dds-header) looks like this:

{{<code language="cpp" title="DDS Header">}}
struct DDS_Header {
	uint32_t signature;
	uint32_t headerSize;
	uint32_t flags;
	uint32_t height;
	uint32_t width;
	uint32_t pitchOrLinearSize;
	uint32_t depth;
	uint32_t mipMapCount;
	uint32_t _unused0[11];
	uint32_t format;
	uint32_t caps1;
	uint32_t caps2;
	uint32_t _unused1[3];
};
{{</code>}}

<div style="display: flex; flex-wrap: wrap; margin-bottom: calc(var(--line-height) * -1.2)">

<section style="flex: 1; padding-bottom: 0">

<p style="margin-top: 0">DDS files are fairly straightforward when storing basic textures.</p>

- `signature` is always the fourCC code `"DDS\0"`.
- `headerSize` is always 124 bytes.
- `flags` is poorly named, essentially stores a flag for every field of the header with valid data.
- `height` and `width` store the dimensions of the base image (the first mip level).
- `pitchOrLinearSize` stores the pitch (number of bytes per scan line) in uncompressed textures, or the size in bytes of
  the first mip level in compressed textures.
- `depth` stores the size of the image in the third dimension (number of layers of a volume texture), if the texture is
  not a volume texture it may not be set to a valid value.
- `mipMapCount` is self-explanatory, the number of mipmaps stored in the file. Mipmaps are images shrunken by a power of
  two from the image that came before it.
- `format` is the format of the image data stored in the file.
- `caps1` and `caps2` together define the type of the stored texture (volume texture? cubemap texture?).

Immediately following this header is the image data. The entire file looks something like this image.

</section>

{{<image src="fig_dds.webp" alt="A diagram of the structure of a DDS file. The header is at the beginning of the file, and immediately following it are each mip level of the texture, going from largest to smallest." style="margin-top: 0; margin-left: 16px; object-fit: contain">}}

</div>

## VTF

DDS is a nice generic format, but its generic-ness made it unsuited for Valve's purposes. Valve needs to store several
things in their textures that the stock DDS header doesn't make room for, e.g. a reflectivity vector for faster
radiosity calculations.

### Early VTF (v7.0-7.2)

VTF may have gone through several internal revisions before the first format we know of publicly, since the major
version (yes, major, there are two version integers) has always been set to 7. The minor version is what gets updated
when the format changes, and it starts at 0. This is what the first public iteration of VTF's header looks like.

{{<code language="cpp" title="VTF Header">}}
#pragma pack(push, 16)

struct VTF_Header_70 {
	uint32_t signature;
	uint32_t majorVersion;
	uint32_t minorVersion;
	uint32_t headerSize;
	uint16_t width;
	uint16_t height;
	uint32_t flags;
	uint16_t frameCount;
	uint16_t startFrame;
	uint32_t _padding0;
	float reflectivity[3];
	uint32_t _padding1;
	float bumpMapScale;
	uint32_t format;
	uint8_t mipCount;
	uint32_t thumbnailFormat;
	uint8_t thumbnailWidth;
	uint8_t thumbnailHeight;
};

// Bitwise identical
struct VTF_Header_71 : public VTF_Header_70 {};

#pragma pack(pop)
{{</code>}}

It doesn't take a genius to figure out Valve copied Microsoft's homework.
Some of the fields are reordered, but in general the contents of the header flow the same direction.
Note that this struct is 16-byte aligned, which is why the reflectivity vector has that weird padding around it.

- `signature` is always the fourCC code `"VTF\0"`.
- `majorVersion` is always 7.
- `minorVersion` ranges from 0 to 6.
- `headerSize` is the same as the DDS `headerSize`, except this header is smaller since it doesn't have tons of reserved
  space.
- `width` and `height` is the same as in DDS, the width and height of the base mip.
- `flags` is actually different than DDS, and is more what you'd expect from a field named flags. It stores flags. See
  [Appendix A](#appendix-a-vtf-flags) for more information on supported flags.
- `frameCount` is the number of "frames", and `startFrame` is the first frame in the animation sequence. Again, more on
  this later!
- `reflectivity` stores the overall reflectivity of the texture, to be used in radiosity calculations. Valve's method of
  calculating it is apparently adding all the pixels' RGB values together, then dividing by the number of pixels. The
  reflectivity is stored in RGB order.
- `bumpMapScale` controls the intensity of the bump map, if this texture is a bump map. It can be ignored otherwise.
- `format` is the same as in DDS, the format of the image data stored in the file. See [Appendix B](#appendix-b-supported-image-formats)
  for more information on supported format types.
- `mipCount` is renamed from `mipMapCount` in DDS.
- `thumbnailFormat`, `thumbnailWidth`, and `thumbnailHeight` are for the VTF thumbnail. More on this later.

Thumbnail data is stored immediately after the header, followed by the image data.

The thumbnail is a low-resolution copy of the base image, which the engine uses in a couple places when it needs to know
general information about the brightness of the texture. It needs to know this information in 2D space, so using the
reflectivity vector doesn't work. The thumbnail format should always be treated as `DXT1` even if the format field is
different, because there are a few official VTFs that have the wrong format in this field, and every VTF creation tool
will create the thumbnail with the `DXT1` format. For VTFs created using Valve's own tooling, the size of the thumbnail
is always observed to be 16x16, as long as the image is square. If the image is not square, the thumbnail size will be
proportional to the image dimensions, with the largest dimension being 16 (so if the image is 2048x1024, the thumbnail
will be 16x8). This behavior is not a hard requirement, and thumbnails that have a different aspect ratio from their
source image will still work. Custom sized thumbnails should also work, but there's no need to go higher, or deviate
from what Valve's official tooling produces.

{{<figure src="fig_thumb.webp" alt="A diagram of the base image being used to create the thumbnail and the reflectivity vector." position="center" caption="The base image is used to calculate both the thumbnail and the reflectivity vector." captionPosition="center">}}

The `flags` field controls several things, although it's also used by `vtex`, Valve's texture compiler, to store data
about the VTF while it's being created. Some flags only have meaning in `vtex`, and some flags only have meaning in the
engine. The list of flags seems to change slightly every VTF version, but one important flag is the cubemap flag
(`2^14`). When this flag is enabled, the VTF has either 6 or 7 faces, the first 6 forming a cubemap and the 7th either
being garbage data or a spheremap when present. When the cubemap flag is present VTF v7.0 has 6 faces, while v7.1 has 7
faces. Unfortunately, Valve has broken their own rules in regards to the presence of the spheremap in official VTFs in
the past, so the only way to know for sure if the spheremap is or isn't present is to check the size of the VTF's image
data. Fortunately, if the VTF is outside the version range v7.1-7.4 (inclusive), we know for sure the spheremap cannot
exist.

The `frameCount` field controls the number of frames the VTF stores. Frames are used for animated textures, and the
frame currently rendered by the game is controlled by the material using the texture. The `startFrame` is the default
value for the current frame (not very complicated).

VTF v7.0-7.1 can't store volumetric textures, despite the fact that DDS can. They might not have cared to implement
support here, since volumetric textures aren't used anywhere in Valve-published Source engine games (or really anywhere
else as far as I can tell). VTF v7.2 adds support for volumetric textures (the only change to the format). At this point
in the timeline we're post-Half-Life 2 release, so they might have been doing internal experiments that would
necessitate this change.

{{<code language="cpp" title="VTF Header with Depth">}}
#pragma pack(push, 16)

// All the fields from v7.1 verbatim, plus...
struct VTF_Header_72 : public VTF_Header_71 {
	uint32_t depth;
};

#pragma pack(pop)
{{</code>}}

<div style="display: flex; flex-wrap: wrap; margin-bottom: calc(var(--line-height) * -1.2)">

<section style="flex: 1; margin-top: calc(var(--line-height) * -1.2)">

Now that volumetric textures are in play, we can properly visualize how image data is ordered.

<!-- 864px is hardcoded as the maximum content width, divide by 2 -->
<div style="max-width: 432px">

```json
"mipmaps" // 0 to mipCount
{
	"frames" // 0 to frameCount
	{
		"faces" // 0 to number of faces
		{       // (either 1, 6, or 7)
			"slices" // 0 to depth
			{
				image data
			}
		}
	}
}
```

</div>

One interesting thing to note is that mipmaps are stored in reverse compared to DDS, going **smallest to largest**. I
don't know for sure, but my theory is they made this change so less of the file would need to be loaded when loading
smaller mip levels from disk. I also don't think the order would matter too much these days.

Another important consideration is that 3D cubemap textures (cubemaps with a nonzero value for depth) are technically
possible to create, but are treated as invalid by the engine. Thus, the face count and the image depth cannot be greater
than 1 simultaneously.

</section>

{{<image src="fig_vtf.webp" alt="A diagram of the structure of a VTF file. The header is at the beginning of the file. Immediately following it is the thumbnail, followed by each mip level of the texture, going from smallest to largest" style="margin-top: 0; margin-left: 16px; object-fit: contain">}}

</div>

### Modern VTF (v7.3-7.5)

VTF v7.3 is where Valve starts to pull away from copying DDS, and attempt to make it a nicer container for more kinds of
texture metadata by introducing the resource system.

{{<code language="cpp" title="VTF Header with Resources">}}
#pragma pack(push, 16)

// All the fields from v7.2 verbatim, plus...
struct VTF_Header_73 : public VTF_Header_72 {
	uint8_t _padding1[3];
	uint32_t resourceCount;
	uint8_t _padding2[8];
};

// Bitwise identical
struct VTF_Header_74 : public VTF_Header_73 {};

// Bitwise identical
struct VTF_Header_75 : public VTF_Header_74 {};

#pragma pack(pop)

struct VTF_Resource {
	uint8_t type[3];
	uint8_t flags;
	uint32_t data;
};
{{</code>}}

Resources are stored after the VTF header in the form of the given struct. Each resource entry counts toward the overall
`headerSize` as well. The type is a three character code used to identify the resource. There is currently only one
resource flag (`2^2`), controlling the location of the resource's data. If this flag is *not* present, the resource data
field holds an absolute offset to the data in the VTF file. If the flag *is* present, the resource data field holds the
resource's data. This is more efficient for resources that only need to store four bytes of data or less. On non-console
platforms the maximum amount of resources in a VTF is 32, but there quite literally are not enough resource types to
ever hit this maximum.

The only required resource in a VTF file is the image data. Thumbnails are technically not necessary to load the VTF,
although they're highly recommended to always include.

The thumbnail and image data are now both considered "legacy" resources, appearing in the resource entry list but
working exactly as they did before. The only difference is their beginning position in the file, which is now controlled
by the resource data field, storing the offset to the thumbnail or image data respectively. The non-legacy resources, if
not stored in the header, store a `uint32_t` just before their data, holding the length of the resource data (including
this integer).

Note that due to overzealous optimization, resource entries in the header need to be sorted from lowest numeric resource
ID to highest numeric resource ID. I found this out the hard way, and you may not even run into any issues writing
unsorted resource entries until a VTF starts to break out of nowhere. Resource entries can be written in any order for
VTFs used by the Strata Source engine branch.

This is the full list of VTF resources as of VTF v7.5, sorted lowest to highest.

| Resource Name  | Resource ID | Stored in Header? | Description of Data |
|---|:---:|:---:|---|
| Thumbnail | `"\x01\0\0"` (1)  | No | "Legacy" resource, holds thumbnail data. |
| Particle Sheet | `"\x10\0\0"` (16)  | No | If the VTF was created using a particle sheet, this is the particle sheet it used (an SHT file). For the sake of focusing on VTF I will save this format for a possible future post. |
| Image Data | `"\x30\0\0"` (48) | No | "Legacy" resource, holds image data. Required in all VTFs from v7.3+. |
| CRC | `"CRC"` (4411971) | Yes | Holds the CRC32 checksum of the *input file* that `vtex` used to create the VTF, not the VTF itself. Unused by the engine. |
| Texture LOD Control Info | `"LOD"` (4476748) | Yes | Holds two `uint8_t` integers, the last two bytes of the data field are unused. Controls the highest loaded mipmap when texture detail is set to "High". For example, if U and V are 11, the highest loaded mipmap on "High" will be 2048x2048 (`2^11`), and on "Very High" the highest loaded mipmap will be 4096x4096 (`2^(11+1)`). There is very little point to this resource's existence in my opinion. |
| KeyValues Data | `"KVD"` (4478539) | No | Holds a string which is *not* null-terminated, specified to be valid KeyValues 1 data. Unused by the engine, is used in third-party tooling to add various information like the texture author and creation date. |
| Extra Texture Flags | `"TSO"` (5198676) | Yes | Stores extra texture flags, which can be used in mods for game-specific purposes. Unused by the engine in official Valve games, unknown if it's used in other games. I have never encountered a VTF in the wild with this resource. |

As for differences between VTF v7.3, v7.4, and v7.5, v7.3 and v7.4 are functionally identical. As previously stated
spheremaps were deprecated in v7.4 and removed in v7.5. v7.5 is unsupported in most Source engine branches before Alien
Swarm, but since it's nearly identical to v7.4, changing the version in the header and adding a dummy spheremap if the
texture is a cubemap is enough to convince the engine to load it.

### Strata Source VTF (v7.6)

Strata Source, a community-maintained fork of the engine, has made some additions to the format to support CPU
compression (usually in tandem with GPU compression), resulting in a new version of the format. VTF v7.6's header is
identical to VTF v7.5.

{{<code language="cpp" title="VTF Header with Compression">}}
#pragma pack(push, 16)

// Bitwise identical
struct VTF_Header_76 : public VTF_Header_75 {};

#pragma pack(pop)
{{</code>}}

Where v7.6 differs is in the new `"AXC"` resource, which is optional and sandwiched between the `"CRC"` and `"LOD"`
resources in the resource entry list.

| Resource Name  | Resource ID | Stored in Header? | Description of Data |
|---|:---:|:---:|---|
| Auxiliary Compression Info | `"AXC"` (4413505) | No | Stores the compression type and strength, as well as the compressed sizes of each individual image. See the description below. |

If the `"AXC"` resource is present, the image data is compressed. The original implementation of the resource looks like
this.

{{<code language="cpp" title="First Public `\"AXC\"` Resource Iteration">}}
struct AXC_V1 {
	// Excluding the size integer present at the beginning of every
	// non-legacy resource's data when not stored in the header
	uint32_t compressionStrength;
	uint32_t compressedSizes[mipCount * frameCount * faceCount];
};
{{</code>}}

This version of the `"AXC"` resource always uses the [Deflate](https://en.wikipedia.org/wiki/Deflate) compression method. The compression
strength is unnecessary at runtime, and stores the level of compression used when creating the texture. For Deflate this
value can be between -1 and 9, inclusive. If the compression strength is 0, the rest of the resource is ignored, since 0
means no compression took place.

The compressed sizes store the size in bytes of every image that the VTF holds. This is typically calculated from the
image dimensions and format, but compression size is non-deterministic so it must be stored. The compressed size layout
is as follows. Note that if the texture is a volumetric texture, the entire 3D texture at the given mip, frame, and face
level is compressed in one block.

```json
"mipmaps" // 0 - mipCount
{
	"frames" // 0 - frameCount
	{
		"faces" // 0 - number of faces
		{       // (either 1, 6, or 7)
			compressed image data length
		}
	}
}
```

Given this information, it is possible to access the compressed size of an image at a given mip, frame, and face level
by using the following formula.

{{<code language="cpp" title="Compressed Image Size Formula">}}
uint32_t compressedSizeAt(AXC_Old axc, uint8_t mip, uint16_t frame, uint8_t face) {
	return axc.compressedSizes[
		(mipCount - mip - 1) * frameCount * faceCount +
	    frame * faceCount +
		face
	];
}
{{</code>}}

When a VTF is compressed, the position of an image at a given mip, frame, and face level relative to the beginning of
the image data resource can be found by summing the sizes of all the compressed images that came before it. The image
data is tightly packed (if it wasn't compression would be rather pointless).

---

A few years after the introduction of VTF v7.6, I updated it to support [Zstd](https://en.wikipedia.org/wiki/Zstd) compression in addition to Deflate.
From my testing Zstd doesn't have a significant advantage in compression size, but it does have a significant advantage
in decompression speed. The new `"AXC"` resource looks like this.

{{<code language="cpp" title="Latest Public `\"AXC\"` Resource Iteration">}}
struct AXC_V2 {
	// Excluding the size integer present at the beginning of every
	// non-legacy resource's data when not stored in the header
	uint16_t compressionStrength;
	uint16_t compressionMethod;
	uint32_t compressedSizes[mipCount * frameCount * faceCount];
};
{{</code>}}

By splitting the compression strength into two shorts, since compression strength was only ever used to store a value
between -1 and 9 inclusive, and VTF is stored in little endian, we can take advantage of the empty space to store the
compression method in a backwards-compatible way. There are three accepted value ranges for `compressionMethod`.

- <=0: Identifies the first iteration of AXC, using Deflate compression.
- 8: The latest iteration of AXC, using Deflate compression.
- 93: The latest iteration of AXC, using Zstd compression.

Note that since the compression strength value is useless at runtime, third-party tooling expecting the old `"AXC"`
resource will continue to work with new Deflate-compressed VTFs. Programs expecting the new `"AXC"` resource should also
load the old AXC resource correctly assuming they handle `compressionMethod` correctly. For these reasons the resource
is not versioned, as making a separate version would actually do more harm than good.

Although `minizip-ng` is not used in the engine or in any third-party tooling I've seen, `compressionMethod` copies its
defines for compression method types. The door is left open for other compression methods, although there's not much of
a point in adding new ones.

## Unspoken Requirements / Things to Know

A few VTF requirements aren't immediately obvious, and sometimes requirements that people will tell you about don't
actually exist. Let's go over all of them.

#### Texture Dimensions

Some say a VTF's image dimensions must not stray from the path of the powers of two. They are utter fools, too cowardly
to taste the sweet forbidden fruit of non-PO2 dimensions. Non-PO2 sized textures may not have worked well in the past,
but in modern times on modern hardware they are fine, and don't seem to cause any issues in Source from my testing. Use
non-PO2 textures sparingly, but don't be afraid of them if they're convenient. (Side note, the dimension for a mip level
below an image that has a dimension that doesn't divide evenly by 2 will be the result of the division plus one (rounded
up). For example, if an image has dimensions 111x64, the mip level below that image is expected to have dimensions
56x32.)

That being said, if a VTF is using a compressed format such as `DXTn`, `BCn`, or `ATIxN` its dimensions must be a
*multiple* of 4. Compressed formats require 4x4 pixel blocks throughout the image. This is also why mip levels lower
than 4x4 sometimes look very weird for compressed formats.

#### Spheremaps

Spheremaps are required for VTF cubemaps with a version between v7.1-7.4 inclusive, but they are never used by the
engine. If you are making a program to create cubemap VTFs feel free to leave the spheremap blank, or insert a doodle.

#### Flags

- When creating a VTF with no mipmaps, the `NO_MIP` and `NO_LOD` flags should be applied.
- When creating a VTF with a format that supports transparency, either the `ONE_BIT_ALPHA` flag or the `MULTI_BIT_ALPHA`
  flag must be applied.
- When creating a cubemap VTF, the `ENVMAP` flag should be applied.

See [Appendix A](#appendix-a-vtf-v75-76-flags) for more information on flags.

## Appendix A: VTF v7.5-7.6 Flags

Most VTF flags are either internal to `vtex` or fairly esoteric. There are a few important ones though. I've left out
the ones that only exist for `vtex`, but kept the ones that are plain weird since the engine likely uses them.

Most of these flags exist for earlier VTF versions, but I don't know which ones and I want to be done writing this post.
Sorry!

| Flag | Value | Description |
|---|:---:|---|
| `POINT_SAMPLE` | `2^0` | Disable texture filtering when sampling from the texture. Obsolete on modern Source branches now that materials can control this as well. |
| `TRILINEAR` | `2^1` | Always use trilinear filtering when sampling from the texture. |
| `CLAMP_S` | `2^2` | Do not wrap on the X axis. |
| `CLAMP_T` | `2^3` | Do not wrap on the Y axis. |
| `ANISOTROPIC` | `2^4` | Always use anisotropic filtering when sampling from the texture. |
| `HINT_DXT5` | `2^5` | Unsure. The Valve Developer Wiki says it's used in skyboxes to remove visible seams, but what a weird name for that. |
| `SRGB` | `2^6` | Texture uses the sRGB color space (sRGB to linear gamma correction will be applied on the GPU). |
| `NORMAL` | `2^7` | Texture is a normal map. |
| `NO_MIP` | `2^8` | Texture has no mipmaps. Should be present on every texture with 1 mip level. |
| `NO_LOD` | `2^9` | Texture has no LOD (it does not use lower mip levels than the base). This flag is utterly useless and should be present when `NO_MIP` is present. |
| `LOAD_LOWEST_MIPS` | `2^10` | Allows the game to load mip levels with dimensions below 32x32. Should only be used for uncompressed formats in my opinion, because compressed formats typically have broken mips below 4x4. |
| `PROCEDURAL` | `2^11` | Texture was either created in code, or will be modified in code. |
| `ONE_BIT_ALPHA` | `2^12` | Should be set for all VTFs storing texture data with one bit alpha. |
| `MULTI_BIT_ALPHA` | `2^13` | Should be set for all VTFs storing texture data with more than one bit of alpha. |
| `ENVMAP` | `2^14` | Set if the VTF is storing a cubemap. |
| `NO_DEBUG_OVERRIDE` | `2^17` | Unknown. |
| `SINGLE_COPY` | `2^18` | Unknown. |
| `NO_DEPTH_BUFFER` | `2^23` | Unknown. |
| `CLAMP_U` | `2^25` | Do not wrap on the Z axis. |
| `VERTEX_TEXTURE` | `2^26` | Texture is a vertex texture for `VertexLitGeneric` materials. |
| `SSBUMP` | `2^27` | Texture is an ssbump. |
| `BORDER` | `2^29` | Clamp to the texture border color on all axes. |

## Appendix B: Supported Image Formats

Unfortunately the list of supported formats is not tied to a VTF version, rather it is tied to the the branch of the
Source engine you're using. Fortunately the list of formats common to all branches is fairly extensive. Check
[the Valve Developer Wiki](https://developer.valvesoftware.com/wiki/VTF_(Valve_Texture_Format)) for a listing with more format-specific information.

SDK2013 is a bit of a dead end in regards to formats, every engine branch created after Alien Swarm should support Alien
Swarm's extra formats in theory (replacing SDK2013's formats).

This listing also does not include console-specific formats.

### All Branches

| Format | ID | Extra Information |
|---|:---:|---|
| `RGBA8888` | 0 |
| `ABGR8888` | 1 |
| `RGB888` | 2 |
| `BGR888` | 3 |
| `RGB565` | 4 |
| `I8` | 5 | Greyscale, luminance |
| `IA88` | 6 | Greyscale, luminance with alpha |
| `P8` | 7 | Uses a palette. Unclear how a palette would be specified, likely a holdover from GoldSrc. I haven't observed this format in any official or unofficial VTFs. |
| `A8` | 8 |
| `RGB888_BLUESCREEN` | 9 | Identical to `RGB888`, except when the color is `0x0000FF` the color is interpreted as transparent instead. |
| `BGR888_BLUESCREEN` | 10 | Identical to `BGR888`, except when the color is `0xFF0000` the color is interpreted as transparent instead. |
| `ARGB8888` | 11 |
| `BGRA8888` | 12 |
| `DXT1` | 13 |
| `DXT3` | 14 |
| `DXT5` | 15 | 
| `BGRX8888` | 16 |
| `BGR565` | 17 |
| `BGRX5551` | 18 |
| `BGRA4444` | 19 |
| `DXT1_ONE_BIT_ALPHA` | 20 |
| `BGRA5551` | 21 |
| `UV88` | 22 |
| `UVWQ8888` | 23 |
| `RGBA16161616F` | 24 |
| `RGBA16161616` | 25 |
| `UVLX8888` | 26 |
| `R32F` | 27 |
| `RGB323232F` | 28 |
| `RGBA32323232F` | 29 |

### Extra SDK2013 Formats

| Format | ID | Extra Information |
|---|:---:|---|
| `EMPTY` | 36 | Should not be used in VTFs, specifies a pixel size of 0 bytes. |
| `ATI2N` | 37 | Broken in all Source engine branches except Strata Source. |
| `ATI1N` | 38 | Broken in all Source engine branches except Strata Source. |

### Extra Alien Swarm (and beyond) Formats

| Format | ID | Extra Information |
|---|:---:|---|
| `RG1616F` | 30 |
| `RG3232F` | 31 |
| `RGBX8888` | 32 |
| `EMPTY` | 33 | Should not be used in VTFs, specifies a pixel size of 0 bytes. |
| `ATI2N` | 34 | Broken in all Source engine branches except Strata Source. |
| `ATI1N` | 35 | Broken in all Source engine branches except Strata Source. |
| `RGBA1010102` | 36 |
| `BGRA1010102` | 37 |
| `R16F` | 38 |

### Extra Strata Source Formats

| Format | ID | Extra Information |
|---|:---:|---|
| `R8` | 69 | Identical to `I8`, except only using the red channel instead of greyscale. |
| `BC7` | 70 |
| `BC6H` | 71 | Specifically the signed half-float variant. |
