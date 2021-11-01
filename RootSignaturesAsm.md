## Dissecting shader assembly with different d3d12 root signatures

In this post I'd like to show how changing the root signature can impact the generated shader assembly. The assembly that we will be looking at is the assembly from RDNA2, specifically compiled and taken on an AMD RX6600 XT. The ISA documentation can be found [here](https://developer.amd.com/wp-content/resources/RDNA_Shader_ISA.pdf). I'm going to assume that the reader has a basic understanding of D3D12 root signatures and GCN/RDNA architecture.
The assembly has been extracted by using the live driver disassembly feature from RenderDoc.

The HLSL shader that we will be referencing is a basic pixel shader that outputs a single color from a constant buffer:

```hlsl
cbuffer ColorConstantBuffer : register(b0)
{
  float4 cColor;
};


float4 PSMain() : SV_TARGET0
{
  return cColor;
}
```

We will be going over different types of root signatures and how they change the generated assembly. Lets start with: `D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE`.

### D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE

We start out with a root signature that is composed of one descriptor table of type `D3D12_DESCRIPTOR_RANGE_TYPE_CBV`. Compiling the pixel shader together with the root signature gives us the following assembly:

```
  s_version     UC_VERSION_GFX10 | UC_VERSION_W64_BIT   // 000000000000: B0802004
  s_inst_prefetch  0x0003                               // 000000000004: BFA00003
  s_getpc_b64   s[0:1]                                  // 000000000008: BE801F80
  s_mov_b32     s0, s2                                  // 00000000000C: BE800302
  s_load_dwordx4  s[0:3], s[0:1], null                  // 000000000010: F4080000 FA000000
  v_mov_b32     v0, 0                                   // 000000000018: 7E000280
  s_waitcnt     lgkmcnt(0)                              // 00000000001C: BF8CC07F
  tbuffer_load_format_xyzw  v[0:3], v0, s[0:3], 0 idxen format:[BUF_FMT_32_32_32_32_FLOAT] // 000000000020: EA6B2000 80000000
  s_waitcnt     vmcnt(0)                                // 000000000028: BF8C3F70
  v_cvt_pkrtz_f16_f32  v0, v0, v1                       // 00000000002C: 5E000300
  v_cvt_pkrtz_f16_f32  v2, v2, v3                       // 000000000030: 5E040702
  exp           mrt0, v0, v0, v2, v2 done compr vm      // 000000000034: F8001C0F 00000200
```

I'll highlight the important bits that are relevant for our constant buffer load.

```
  s_getpc_b64     s[0:1]                                  // 000000000008: BE801F80
  s_mov_b32       s0, s2                                  // 00000000000C: BE800302
  s_load_dwordx4  s[0:3], s[0:1], null                    // 000000000010: F4080000 FA000000
```

The `s_getpc_b64` is partially used to get the memory address for the descriptor table. Both the shader and descriptor table live in the same address space. The compiler makes use of this and stores the top 32 bits in s1. My assumption is that s2 contains the lower 32 bits of the descriptor table that holds the constant buffer descriptor. Eventually `s_load_dwordx4` loads the constant buffer descriptor from the descriptor table defined in s[0:1] at index 0 into s0 through s3.

```
  tbuffer_load_format_xyzw  v[0:3], v0, s[0:3], 0 idxen format:[BUF_FMT_32_32_32_32_FLOAT] // 000000000020: EA6B2000 80000000
```

With `tbuffer_load_format_xyzw` we load the `cColor` value into v0 through v3 using the constant buffer descriptor we loaded earlier.
It's interesting to see that the compiler decides to use a `tbuffer_load_format_xyzw` instead of a `s_buffer_load_dwordx4` !check_this_plz! instruction even when the constant buffer value is uniform across the wave. My guess is that it's an optimization related to the color export. Changing the return to `return cColor + float4(1, 1, 1, 1);` actually turns the load into `s_buffer_load_dwordx4`.
This is the assembly that you get when you use a D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE as a root signature entry. Now lets see what happens if we change it to `D3D12_ROOT_PARAMETER_TYPE_CBV`.

### D3D12_ROOT_PARAMETER_TYPE_CBV

Again, we use the same shader but this time we use D3D12_ROOT_PARAMETER_TYPE_CBV instead. On the CPU side we need to switch from building up a descriptor table to using `ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView`. Compiling with the updated root signature, we get the following:

```
  s_version     UC_VERSION_GFX10 | UC_VERSION_W64_BIT   // 000000000000: B0802004
  s_inst_prefetch  0x0003                               // 000000000004: BFA00003
  v_mov_b32     v0, 0                                   // 000000000008: 7E000280
  s_and_b32     s0, s3, lit(0x0000ffff)                 // 00000000000C: 8700FF03 0000FFFF
  s_mov_b32     s3, lit(0x2104bfac)                     // 000000000014: BE8303FF 2104BFAC
  s_or_b32      s0, s0, lit(0x00100000)                 // 00000000001C: 8800FF00 00100000
  s_mov_b32     s1, s0                                  // 000000000024: BE810300
  s_mov_b32     s0, s2                                  // 000000000028: BE800302
  s_movk_i32    s2, 0x1000                              // 00000000002C: B0021000
  tbuffer_load_format_xyzw  v[0:3], v0, s[0:3], 0 idxen format:[BUF_FMT_32_32_32_32_FLOAT] // 000000000030: EA6B2000 80000000
  s_waitcnt     vmcnt(0)                                // 000000000038: BF8C3F70
  v_cvt_pkrtz_f16_f32  v0, v0, v1                       // 00000000003C: 5E000300
  v_cvt_pkrtz_f16_f32  v2, v2, v3                       // 000000000040: 5E040702
  exp           mrt0, v0, v0, v2, v2 done compr vm      // 000000000044: F8001C0F 00000200
```

Our load for the descriptor table is gone and has been replaced by a bunch of moves and bit twidling. We have to take a slight step back to understand what exactly the compiler decide to do here. 
You could think of a buffer descriptor as a struct where members take up a specific bit range. For example the buffer descriptor could look something like this if coded in C++:

```cpp
struct BufferDescriptor
{
  ...
  uint stride : 4;
  uint num_elements : 16;
  uint format : 6;
  uint memory_address : 20;
  ...
};
```

The descriptor essentially tells the gpu where to find all the relevant information when reading/writing a buffer. The shader itself already contains a lot of the information the compiler needs to read the constant buffer. But one thing it doesn't have (which is quite important) is the memory address where to actually read the constant buffer data from. We gave this address when calling SetGraphicsRootConstantBufferView.

What the compiler is doing with `D3D12_ROOT_PARAMETER_TYPE_CBV` is building up the constant buffer descriptor inline with a bit of ALU work:

```
  s_and_b32     s0, s3, lit(0x0000ffff)                 // 00000000000C: 8700FF03 0000FFFF
  s_mov_b32     s3, lit(0x2104bfac)                     // 000000000014: BE8303FF 2104BFAC
  s_or_b32      s0, s0, lit(0x00100000)                 // 00000000001C: 8800FF00 00100000
  s_mov_b32     s1, s0                                  // 000000000024: BE810300
  s_mov_b32     s0, s2                                  // 000000000028: BE800302
  s_movk_i32    s2, 0x1000                              // 00000000002C: B0021000
  tbuffer_load_format_xyzw  v[0:3], v0, s[0:3], 0 idxen format:[BUF_FMT_32_32_32_32_FLOAT] // 000000000030: EA6B2000 80000000
```

The address of the constant buffer is loaded up into register s3 before the wave is launched. The compiler makes sure it only adds the relevant bits to the constant buffer descriptor by masking out the bits we should not overwrite. For example it decides to only take the bottom 16 bits of whatever is stored in s3 with `s_and_b32 s0, s3, lit(0x0000ffff)`, this would translate to c++ code as `s0 = s3 & 0x0000ffff`.

By switching to `D3D12_ROOT_PARAMETER_TYPE_CBV` we are able to remove one indirection, the descriptor table lookup. We traded a buffer load for a bit of ALU work. In most cases this would be faster, because waiting on memory is generally slow. Could we do even better? Enter: `D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS`.

### D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS

And again, we change our root signature to use `D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS`. This time we need to use `ID3D12GraphicsCommandList::SetGraphicsRoot32BitConstants` to set the constant buffer values. We make the change and hit compile:

```
  s_version     UC_VERSION_GFX10 | UC_VERSION_W64_BIT   // 000000000000: B0802004
  s_inst_prefetch  0x0003                               // 000000000004: BFA00003
  v_cvt_pkrtz_f16_f32  v0, s2, s3                       // 000000000008: D52F0000 00000602
  v_cvt_pkrtz_f16_f32  v1, s4, s5                       // 000000000010: D52F0001 00000A04
  exp           mrt0, v0, v0, v1, v1 done compr vm      // 000000000018: F8001C0F 00000100
```

Holy cow, did our shader just turn into 5 lines of assembly? This can't be it right? Nope, entirely expected behaviour :)

What happened is that our constant buffer data is directly loaded into scalar registers s[2:5] before the wave is launched. All that the shader needs to do is read those registers to get the values. That's it. Doesn't get much faster than this.

But please don't start switching to `D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS` everywhere. While it can definietly help in some cases, it depends a lot on the use case. Ideally you want to store data in there that is frequently accessed but also doesn't have a giant memory footprint. Root constants take up a considerable amount of data in the root signature (see [Root Argument Limits](https://microsoft.github.io/DirectX-Specs/d3d/ResourceBinding.html#root-argument-limits)), on top of that the compiler also needs to reserve scalar registers in order to store the data. You could have cases where you are better of using those scalar registers somewhere else. Only way to find out is to profile before making such changes.

### Pros and Cons

While the descriptor table results in a memory load, generally it's the safest option to pick. They allow you to store way more descriptors then inline descriptors or root constants.

Inline descriptors are limited to [buffer resources](https://microsoft.github.io/DirectX-Specs/d3d/ResourceBinding.html#using-descriptors-directly-in-the-root-arguments) and cannot be used for textures. There is also no bounds checking happening for inline descriptor. The shader is now really responsible for not fetching out of bounds. 

Root constants have the least amount of indirection but also take up scalar registers. Depending on your shader they might actually be usefull for other parts of your code. Having scalars spill to vgpr's is no fun either. The same applies here that no bounds checking is done and an out of bounds read will produce undefined results.

As always profile when making these types of changes, that's the only way to know for sure if things will be faster :)

### The end

I hope this post gave a better understanding on how root parameter types translate to different concepts in assembly. 

If you made it to the end, thank you for reading :) Otherwise hi mom!
