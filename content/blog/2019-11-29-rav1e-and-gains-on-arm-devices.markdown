+++
date = "2019-11-29"
title = "rav1e and gains on ARM Devices"
description = "Vibhoothi"
menu = "blog"
weight = 996
+++


Hi, 

It's been a while I have written a blog about the work I am doing lately. So yeah, I have been working on [rav1e](https://github.com/xiph/rav1e), the AV1 Encoder written in rust as part of [Project Iris](https://mindfreeze.videolan.me/iris).  Currently, if we see, there are other Open-Source Encoders available like [libaom](https://aomedia.googlesource.com/aom/) from Google,  [SVT-AV1](https://github.com/OpenVisualCloud/SVT-AV1) from Intel and Netflix. Rav1e’s memory footprint makes it a good starting point for new-cases like Software Encoding in ARM devices, Real-time streaming, while libaom and SVT-AV1 are either too slow or resource-intensive. So it would be much easier to make rav1e fast and power-efficient due to the low-complexity functions.

 In the end, we are getting around ~12-20% Improvement in Encoding Time and FPS which is the first step making adoption of AV1 to Mobile devices. It is also very important to note that we’ve been using what SIMD code we have rather than proceeding in the order of lowest-hanging fruit. The relative gains would be more impressive if the outstanding functions were optimized first. Our priority was proving the infrastructure for merging, testing and benching on ARM Devices feasible and now it's more realistic. 

I would like to express my sincere thanks to barrbrain, lu_zero, krish&#95iyer and unlord for their support from the community which made me directly or indirectly motivate to do these.


## How did we do it? 

So as an initial step, we have to [compile assembly functions](https://github.com/vibhoothiiaanand/as_playground/commit/1d885629c3930119bda1e7a8b8982948cade5965) with rust itself.
For having the Assemby in rav1e we did some things like this:

*  In the [build.rs](https://github.com/xiph/rav1e/blob/master/build.rs) script, we did like this:

   ``` rust
    // @@ +66 @@
    #[cfg(feature = "asm")]
    fn build_asm_files() {
     use std::fs::File;
     use std::io::Write;
    let out_dir = env::var("OUT_DIR").unwrap();
    {
      let dest_path = Path::new(&out_dir).join("config.h");
      let mut config_file = File::create(dest_path).unwrap();
      config_file.write(b" #define PRIVATE_PREFIX rav1e_\n").unwrap();
      config_file.write(b" #define ARCH_AARCH64 1\n").unwrap();
      config_file.write(b" #define ARCH_ARM 0\n").unwrap();
      config_file.write(b" #define CONFIG_LOG 1 \n").unwrap();
      config_file.write(b" #define HAVE_ASM 1\n").unwrap();
    }
    cc::Build::new()
      .files(&[
        "src/arm/64/mc.S",
        
        "src/arm/tables.S",
      ])
      .include(".")
      .include(&out_dir)
      .compile("rav1e-aarch64");
    rerun_dir("src/arm");
    }

    // @@ +123 @@
     #[cfg(feature = "asm")]
    {
     @@ ++ @@
    if arch == "aarch64" {
      println!("cargo:rustc-cfg={}", "asm_neon");
      build_asm_files()
     }
    }
   ``` 

* In the [cpu_features](https://github.com/xiph/rav1e/tree/master/src/cpu_features), we have to add AARch64 target. For that there are some things to be taken care of:
     - How we build the stuff with the cargo build system for modules, it is a bit alien at first since.
     - Making sure the flags are invoked currently when specific architecture is detected.
 
  We have elided `is_aarch64_feature_detected!()` because AArch64 Mandates NEON. We maintain `CpuFeatureLevel` primarily to focus on native Rust kernels at the runtime, in a way consistent with other architecture. 
\
  For AArch64 we have two CPU level, either `NATIVE` or `NEON`, if a user wants to change that, then he could use the environment variable `RAV1E_CPU_TARGET=NATIVE`.
  
  We have made [aarch64.rs](https://github.com/xiph/rav1e/blob/master/src/cpu_features/aarch64.rs) file where AARch64 CPU Detection happens.

   ```rust
   impl Default for CpuFeatureLevel {
   fn default() -> CpuFeatureLevel {
    let detected = CpuFeatureLevel::NEON;
    let manual: CpuFeatureLevel = match env::var("RAV1E_CPU_TARGET") {
      Ok(feature) => CpuFeatureLevel::from_str(&feature).unwrap_or(detected),
      Err(_e) => detected,
    };
    if manual > detected {
      detected
    } else {
      manual
    }
  }
   ```

   Here environment variable overrides the default value if needed.
\

*  In the [mod.rs](https://github.com/xiph/rav1e/blob/master/src/cpu_features/mod.rs), we would require to add to use aarch64.
   So, it would be like 
   ```rust 
   cfg_if::cfg_if! {
   // @@ ++4 @@
   } else if #[cfg(asm_neon)] {
    mod aarch64;
    pub use aarch64::*;
   } else {
    mod native;
    pub use native::*;
    }
  }
   ```


* [Cargo.toml](https://github.com/xiph/rav1e/blob/master/Cargo.toml), we have added `cc-rs` as a build-dependency.
 
    ``` toml
    [features]
    # @@ +20 @@
    asm = ["nasm-rs", "cc"] 
    # @@ +73 @@
    [build-dependencies]
    cc = { version = "1.0", optional = true, features = ["parallel"] }
    ```
    Here we added the `cc` build dependency addition to nasm to a feature `asm` which would help us to have both `x86` and `AArch64` seamlessly with help of `#[cfg]` flag where we would be checking if `AAarch64` / `x86`, where in the future parts we would be adding an initial check with help of `#[cfg(feature = "asm")]`.
   \
    For the assembly files from [dav1d](https://code.videolan.org/videolan/dav1d/tree/master/src/arm), there are the config files which defines how the assembly, should be interpreted, is it `AArch64` only or is it having `HAVE_ASM` and assigning default values for those. Here we are using `PRIVATE_PREFIX` as `rav1e_` in favour of matching how we did with all other functions for `x86` and styling. Using standard input/output of rust (`std::io::Write`, `std::fs::File`) for creating and writing  the config file. To make minimal chances to assembly, we did not touch any of `.S` files in this process, so we did make config files and write with this method. For compiling, we are using cc's `Build` where we would be inputting file as a single element or as an array, we are inputting as array since we would be having multiple files to have and we like to have a single object compiled for those files. We would also explicitly mention the compiler to compile the inside files so we are mentioning that too, include the config file, and most importantly, rerunning the files in that directory because `cc-rs` does not emit, `rerun-if-` lines for sources or ENV variables. So we would require to rebuild if the assembly is modified. There is an open issue ([#230](https://github.com/alexcrichton/cc-rs/issues/230)) too regarding this in `cc-rs` do keep an eye on that.
    
After having these three, the next main thing we would require to do is, having rust wrappers for the Assembly functions.
\
For the assembly functions which we have compiled, we would require to get the symbols. For that, we can use Unix command `objdump` to dump objects and filter out `rav1e` and further sort if needed.
The command which can be used would be like  

`objdump -t target/debug/build/rav1e-02ad44fbebde75fb/out/librav1e-aarch64.a | rg "rav1e"`

```assembly
.....
In archive target/debug/build/rav1e-02ad44fbebde75fb/out/librav1e-aarch64.a:
0000000000000000 g     F .text	0000000000000000 rav1e_avg_8bpc_neon
0000000000000414 g     F .text	0000000000000000 rav1e_w_avg_8bpc_neon
0000000000000a14 g     F .text	0000000000000000 rav1e_mask_8bpc_neon
000000000000121c g     F .text	0000000000000000 rav1e_w_mask_444_8bpc_neon
0000000000001438 g     F .text	0000000000000000 rav1e_w_mask_422_8bpc_neon
000000000000165c g     F .text	0000000000000000 rav1e_w_mask_420_8bpc_neon
000000000000187c g     F .text	0000000000000000 rav1e_blend_8bpc_neon
0000000000001a0c g     F .text	0000000000000000 rav1e_blend_h_8bpc_neon
0000000000000000         *UND*	0000000000000000 rav1e_obmc_masks
0000000000001c14 g     F .text	0000000000000000 rav1e_blend_v_8bpc_neon
00000000000020fc g     F .text	0000000000000000 rav1e_put_8tap_regular_8bpc_neon
....
```

with the objdump we could know the symbols are generated and based upon that, we can write/generate rust specific functions. 
When we are creating rust specific wrappers for the assembly to work seamlessly, there should be some guidelines to be followed.
\
So for the rust wrapper, I would take Motion Compensation (`mc.S`) as example reference.
\
The rust function is written in `src/mc.rs` while thee wrapper is written in `src/asm` folder. We are having AArch64 wrappers in aarch64 folder inside src, so `src/asm/aarch64`. 

The file structure should be something like this
```
.
src/
├── arm
│   ├── 64
│   │   ├── mc.S
│   │   └── util.S
│   ├── asm.S
│   └── tables.S
├── asm
│   ├── aarch64
│   │   ├── mc.rs
│   │   ├── mod.rs
│   ├── mod.rs
`-- mc.rs
```

In the `mod.rs` found in asm, we would be having a flag to check if it is having `asm_neon` flag and if that is present use `aarch64` module.
we would be doing like this 
```rust 
#[cfg(asm_neon)]
pub mod aarch64;
```

For the mc.rs, we would be writing accordance to main mc.rs and also for wrapper which we wrote for `x86`. We would be also writing the tests for the respective files.
\
If we see in dav1d's C's implementation ([mc_init_tmpl.c](https://code.videolan.org/videolan/dav1d/blob/master/src/arm/mc_init_tmpl.c#L33)), and assembly implementaiton ([mc.s](https://code.videolan.org/videolan/dav1d/blob/master/src/arm/64/mc.S#1345)) happens like this, while for us, we have a bit more cleaner way to do it, thanks to rust.
```c
decl_mc_fn(dav1d_put_8tap_regular_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_regular_smooth_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_regular_sharp_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_smooth_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_smooth_regular_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_smooth_sharp_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_sharp_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_sharp_regular_8bpc_neon);
decl_mc_fn(dav1d_put_8tap_sharp_smooth_8bpc_neon);
decl_mc_fn(dav1d_put_bilin_8bpc_neon);
```
<center>file: dav1d/src/mc_tmpl.c</center>


```rust
decl_mc_fns!(
  (REGULAR, REGULAR, rav1e_put_8tap_regular_8bpc_neon),
  (REGULAR, SMOOTH, rav1e_put_8tap_regular_smooth_8bpc_neon),
  (REGULAR, SHARP, rav1e_put_8tap_regular_sharp_8bpc_neon),
  (SMOOTH, REGULAR, rav1e_put_8tap_smooth_regular_8bpc_neon),
  (SMOOTH, SMOOTH, rav1e_put_8tap_smooth_8bpc_neon),
  (SMOOTH, SHARP, rav1e_put_8tap_smooth_sharp_8bpc_neon),
  (SHARP, REGULAR, rav1e_put_8tap_sharp_regular_8bpc_neon),
  (SHARP, SMOOTH, rav1e_put_8tap_sharp_smooth_8bpc_neon),
  (SHARP, SHARP, rav1e_put_8tap_sharp_8bpc_neon),
  (BILINEAR, BILINEAR, rav1e_put_bilin_8bpc_neon)
 );
```
<center>file: rav1e/src/asm/aarch64/mc.rs</center>

The assembly which we did import from dav1d(`src/arm/64`) is not modified and untouched. Based on the symbol which we have found from `objdump` from those, we have made relevant symbols which also matches our native rust implementation.
\
In the main `mc.rs` file, we would require one more line to add to make sure that the mc.rs wrapper which we wrote for assembly is taken instead of normal native ones, for that we will further require to add a condition to check 

```rust 
 cfg_if::cfg_if! {
    if #[cfg(nasm_x86_64)] {
      pub use crate::asm::x86::mc::*;
    } else if #[cfg(asm_neon)] {
      pub use crate::asm::aarch64::mc::*;
    } else {
      pub use self::native::*;
    }
  }
```

When we are doing this, we _should_ be really careful with the path to the module since if it is wrong, then the whole program would not work. So the easiest way to find the exact way to call the module would be with the help of UNIX command `pwd`
for `mc.rs` it is
`/home/vibhoothiiaanand/rav1e/src/asm/aarch64/mc.rs`
so it would be 
\
`asm-> aarch64 -> mc-> all the functions` is
\
`asm::aarch64::mc::*;`
\
also, make sure when you do, it should be having `::` and not `:` as it is syntactically wrong.

List of commits which are integrated into rav1e, this list would give you a better idea on how much we did take and how much we did not take.
<details>

```diff
- arm64: looprestoration: Minimal scheduling improvements
- arm64: looprestoration: Fix a typo  …
- arm64: looprestoration: Fix register references in comments
- arm64: looprestoration: Use ld2r instead of ld1+dup+dup
+ arm64: ipred: Make sure all symbols are aligned 
+ arm: util: Split movrel into movrel and movrel_local
- arm64: ipred: NEON implementation of the cfl_ac functions  
+ arm64: ipred: NEON implementation of the cfl_pred functions  
- arm64: ipred: NEON implementation of the filter function  
- arm64: ipred: NEON implementation of palette prediction  
+ arm64: ipred: NEON implementation of smooth prediction  
+ arm64: ipred: NEON implementation of paeth prediction  
+ arm64: mc: Use addp instead of addv+trn1 in warp  
- arm64: cdef: Improve find_dir  
- arm64: cdef: Calculate two initial parameters in the same vector  
- arm64: cdef: Use loads with postincrement in more places in the padding function
- arm64: cdef: Rewrite an expression slightly  
+ arm64: mc: Schedule instructions better in the warp8x8 functions  
+ arm64: mc: Use sbfx instead of ubfx+sxth in the warp function
+ arm64: ipred: NEON implementation of dc/h/v prediction modes  
+ arm64: itx: Fix overflows in idct  
+ arm64: itx: Consistently use the factor 2896 in adst  
+ arm64: itx: Use smull+smlal instead of addl+mul  
+ arm64: itx: Do the final calculation of adst4/adst8/adst16 in 32 bit to avoid too narrow clipping  
- arm64: mc: NEON implementation of w_mask_444/422/420 function  
- arm64: mc: NEON implementation of blend, blend_h and blend_v function  
- Add msac optimizations  
+ arm64: itx: Add NEON optimized inverse transforms  
+ arm64: Consistently name macro arguments tX for temporaries in transposes
- arm64: msac: Add handwritten versions of msac_decode_bool functions  
- arm64: msac: Fix a typo in a comment
- Add __attribute__((cold)) to rarely used functions
- arm64: remove invalid macro argument delimiter
- arm64: msac: Implement NEON msac_decode_symbol_adapt  
- arm64: loopfilter: Implement NEON loop filters  
- arm64: looprestoration: Add a NEON implementation of SGR  
- arm64: cdef: Clarify a slightly confusing comment  
- arm64: cdef: Use a smarter padding constant  
- arm64: cdef: Do saturating subtractions to avoid max operations with 0  
+ fix dav1d spelling
- arm64/ios: use prefixed dav1d_mc_warp_filter symbol
- arm64: mc: NEON implementation of warp8x8{,t}  
- arm64: cdef: NEON implementation of the dir function  
- arm64: cdef: NEON optimized cdef filter function  
- arm64: looprestoration: Optimize loop termination checks in copy_narrow_neon
- arm64: looprestoration: Simplify the horizontal filtering of one pixel at a time
- arm64: looprestoration: Simplify the setup of wiener_filter_v_neon
- arm64: looprestoration: Fix the loop condition in copy_narrow_neon  
- arm64: looprestoration: Fix comment typos
- arm64: looprestoration: Avoid unnecessary alignment of the mid buffer  
+ arm64: mc: Optimize mc_8tap_regular_w4_hv_8bpc for A53  
+ arm64: mc: Simplify the 8tap_2w_hv code slightly  
+ arm64: mc: Optimize the mul_mla_8_* macros for Cortex A53  
+ arm64: mc: Improve a comment
+ arm64: mc: Remove unused/unnecessary macro args
+ arm64: mc: Use ubfx instead of ubfm, for consistency with arm  
- arm64: looprestoration: NEON optimized wiener filter  
- arm64: mc: Implement 8tap and bilin functions
+ build: add support for arm/aarch64 asm and integrate checkasm
- arm64/mc: add 8-bit neon asm for avg, w_avg and mask
+ aarch64: Always use the PIC version of movrel for iOS
+ arm64: Don't use uxth for extending a register
+ arm64: mc: Make the jump tables local symbols
+ rename arch specific bitdepth template files
- arm64: mc: Implement 8tap and bilin functions
- arm64: looprestoration: NEON optimized wiener filter
```

</details>

## What are the gains?? Benchmarks !!

So we have encoded 100 frames of a 720p Clip (4:2:0) [ducks_takes_off](https://media.xiph.org/video/derf/y4m/ducks_take_off_420_720p50.y4m) with rav1e [0.1 release](https://github.com/xiph/rav1e/releases/tag/0.1.0), then the subsequent [weekly release](https://github.com/xiph/rav1e/releases/tag/20191120) and also [master_a6c217b5](https://github.com/xiph/rav1e/tree/a6c217b5cd276161d1c48b7b353d58f0ca29b7c3) on Raspberry Pi 3 B+ which is running Ubuntu 18.04 64 Bit(AArch64) for testing and found some intresting results.



### Encoding Time Gains 


| Release Name          | Encoding Time | Percent Gain from Previous Release | Percent gain from 0.1 Release  |
|-----------------------|---------------|------------------------------------|--------------------------------|
| rav1e_0.1.0_ 97538d1  | 01:22:47      | NA                                 | NA                             |
| rav1e_weekly_ 7344dae | 01:19:05      | 4.469498691                        | 4.469498691                    |
| rav1e_master_a6c217b5 | 01:00:59      | 22.88724974                        | 35.74747199                    |

\
![image500c](https://i.imgur.com/3WsGowz.png)
\
From the encoding, we got around 35% improvement, which is really a great first step towards faster AV1 encoding.
### FPS Gains 

| Release Name          | fps  | Percent Gain from Previous Release | Percent gain from 0.1.0 Release |
|-----------------------|------|------------------------------------|---------------------------------|
| rav1e_0.1.0_ 97538d1  | 0.2  | NA                                 | NA                              |
| rav1e_weekly_ 7344dae | 0.21 | 5                                  | 5                               |
| rav1e_master_a6c217b5 | 0.27 | 28.57142857                        | 35                              |

\
\
![image500c](https://i.imgur.com/gBTfwAy.png)
\
We got 35% improvement for FPS in the ARM devices, which is again good.
\
Detailed can be found from this [Google Sheets](https://docs.google.com/spreadsheets/d/1gdhlgVfiscyXoaEFjSisoYE-ta15VY1gVd3QcXuNNFw/edit#gid=370028033).
\
Pastebin for the this: [Logs](https://pastebin.com/pUSteP2R).
\
Also, to mention, Thanks to [u/Toggleton](https://www.reddit.com/user/toggleton/) for his extensive, report. With that, we could get more understanding of how much gain we have achieved.

| File 20Sec 600Frames | Rav1e arm 0.4.0 PR | Rav1e arm 0.5 PR |
|----------------------|--------------------|------------------|
| Time                 | 255m39,731s        | 255m13,358s      |
| FPS                  | 0.039 fps          | 0.039 fps        |
| Speed 6              |                    |                  |
| Time                 | 183m30,313s        | 183m23,849s      |
| FPS                  | 0.054 fps          | 0.055 fps        |
| Speed 9              |                    |                  |
| Time                 | 168m18,575s        | 168m6,684s       |
| FPS                  | 0.059 fps          | 0.059 fps        |

Input file: [https://storage.cloud.google.com/ugc-dataset/original_videos/Gaming/1080P/Gaming_1080P-6dc6.mkv](https://storage.cloud.google.com/ugc-dataset/original_videos/Gaming/1080P/Gaming_1080P-6dc6.mkv)
\
Preview: [https://storage.googleapis.com/ugc-dataset/previews/Gaming_1080P-6dc6.mp4](https://storage.cloud.google.com/ugc-dataset/original_videos/Gaming/1080P/Gaming_1080P-6dc6.mkv)
\
Logs: [https://pastebin.com/0Uz1bkK4](https://pastebin.com/0Uz1bkK4)
\
Log for speed-2(0.4 PR): [https://pastebin.com/xdvmw6Sb](https://pastebin.com/xdvmw6Sb)


|File 20Sec 600Frames|0.1(last benchmark)|Changes 0.1 - master(without 0.4)|Rav1e Master|Rav1e arm 0.4.0 PR|Change for this PR|
:--|:--|:--|:--|:--|:--|
|**Speed 2**||||||
|Time|307m0,406s|5,43%|290m34,193s|255m39,731s|12,04%|
|FPS|0.033 fps||0.034 fps|0.039 fps||
|**Default Speed 6**||||||
|Time|217m63s|11,91%|194m46,607s|183m30,313s|5,74%|
|FPS|0.046 fps||0.051 fps|0.054 fps||
|**Speed 9**||||||
|Time|189m21,137s|7,33%|175m34,756s|168m18,575s|4,08%|
|FPS|0.053 fps||0.057 fps|0.059 fps||


Also to add upon this gains, this ~35% gain, is not solely due to this, we also had general speedups by refactoring codebase and much more during the Workweek in Tokyo during VideoLAN Developer Days 2019. So we could say, we got at least more than 20% due to the inclusion of Assembly.


### Future plans

Currently, we have only imported all functions and integrated a few of them and we got this much gains. So next plans which we have are also promising like this 
- Implement SAD and SATD functions, which makes x86 and AArch64 assembly equal.
- Fix the CDEF bug which we are facing. The bug which we are facing is, the offset is having a one-bit difference. We have tried to incorporate CDEF changes but it is not so doable easily, we have to rework on memory buffers to incorporate changeable stride for input buffer which would take time and would be a blocker. So what we are going to do now is, have all x86 integrated changes for AArch64 also and have it pass all tests and do a benchmark.
- Integrate Looprestoration from dav1d.
- Integrate Loopfiltering from dav1d.
- Write better tests for transformations 
- Start writing RDO Assembly functions, RDO is an expensive one in our rav1e.
- Have better user-friendly documentation which would help people to understand AV1 better.


Freely,
\
~ mindfreeze

