## Dissecting shader assembly with different d3d12 root signatures

In this post I'd like to show how changing the root signature can impact the generated shader assembly. The assembly that we will be looking at is the assembly from RDNA2, specifically taken on a AMD RX6600 XT. The ISA documentation can be found [here](https://developer.amd.com/wp-content/resources/RDNA_Shader_ISA.pdf). We are going to assume that the reader has a basic understanding of D3D12 root signatures.
The asm has been extracted by using RenderDoc.

The HLSL shader that we will be referencing is a basic pixel shader that outputs a single color from a constant buffer:

```
cbuffer ColorConstantBuffer : register(b0)
{
  float4 cColor;
};


float4 PSMain() : SV_TARGET0
{
  return cColor;
}
```

### D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE

We start out with a root signature that has one entry for our pixel shader. A D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE with range type set to D3D12_DESCRIPTOR_RANGE_TYPE_CBV. Compiling the pixel shader together with the root signature gives us the following assembly:

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
  s_load_dwordx4  s[0:3], s[0:1], null                  // 000000000010: F4080000 FA000000
```
The `s_getpc_b64` is a clever way of setting s1 to zero, it stores the byte address of the next instruction into s0 and s1. Essentially setting s0 to 00000000000C and s1 to 000000000000


### D3D12_ROOT_PARAMETER_TYPE_CBV

Switching to a `D3D12_ROOT_PARAMETER_TYPE_CBV` gets us:

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

### D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS

Switching to `D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS`:

```
  s_version     UC_VERSION_GFX10 | UC_VERSION_W64_BIT   // 000000000000: B0802004
  s_inst_prefetch  0x0003                               // 000000000004: BFA00003
  v_cvt_pkrtz_f16_f32  v0, s2, s3                       // 000000000008: D52F0000 00000602
  v_cvt_pkrtz_f16_f32  v1, s4, s5                       // 000000000010: D52F0001 00000A04
  exp           mrt0, v0, v0, v1, v1 done compr vm      // 000000000018: F8001C0F 00000100
```
