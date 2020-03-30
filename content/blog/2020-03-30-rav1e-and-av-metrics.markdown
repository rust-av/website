+++
date = "2020-03-30"
title = "Integration of AV-Metrics into rav1e, the AV1 Encoder"
description = "Vibhoothi"
menu = "blog"
weight = 994
+++

Hi,

Hope all are doing well, in this COVID-19 Outbreak and the lockdown, let us take a minute sometime to pray for the people who did pass away and who are under diagnosis with Corona,

So, this blog post aims to provide some technical overview of how [introduction of v-frame](https://github.com/xiph/rav1e/pull/2164), [migration to v-frames for av-metrics](https://github.com/rust-av/av-metrics/pull/67) and also [integration of av-metrics into rav1e](https://github.com/xiph/rav1e/pull/2184) will help for rapid analysis. Rapid analysis like measurement of encoder’s complexity with metrics like PSNR, SSIM, A-PSNR, PSNR-HVS, MS-SSIM, CIEDE2000 etc.
These are based on [IETF NETVC testing draft](https://tools.ietf.org/html/draft-ietf-netvc-testing-09).

So if you are new to AV1 or rav1e or these things, In 2018, the Alliance for Open Media released its next-generation Video Codec AV1. AV1 is so far the most efficient royalty-free Video Codec Standard with respect to VP9, x264. Considering currently available AV1 Open-Source encoders libaom, SVT-AV1 and rav1e.

The Rust AV1 Encoder(rav1e) project by Mozilla, Xiph Organisation is a cleanroom AV1 Implementation having lower memory footprint making it a good starting point while others are either too slow or resource-intensive.

With the help of [AreWeCompressedYet](https://arewecompressedyet.com/) runs and analysis on personal high-performance servers and machines, the post also aims to provide some minor analysis and other insights overtime.

Some notable things from the experimentation are as follows:

- rav1e is always having low complexity compared to SVT-AV1 and libaom since rav1e is having a lean implementation.

     ![image650c](https://i.imgur.com/tvPO11D.png)
     <center><b>Figure 1</b>: General overview of rav1e File-structure</center>
     <br></br>

-  With the project's initial phase of support [ARM(AArch64) Architecture](https://mindfreeze.videolan.me/blog/rav1e-and-gains-on-arm-devices), which made rav1e more optimised with more than 35\% Percent improved Encoding Time and in Frames Per Second(FPS), which is a great boost for having rav1e on multiple platforms, which is first time happening in AV1.

- Currently, rav1e is fastest AV1 Encoder on an ARM Devices when it was tested. The test was carried on Raspberry Pi 3 B+ which houses 1.4GHz Cortex-A53 with 1GB RAM as a baseline for CPU power so when running on powerful AArch64 devices like a mobile, it gives enhanced results.

- Low-latency mode makes the encoding very fast compared to normal encoding at default speed levels, where the trade-off between speed and quality is being compromised. The time between the video frame entering the encoder and the packet containing it is the latency.

- The optimisation is an iterative process. The below diagram explains the general approach of Analysis and optimisation workflow.

    ![image650c](https://i.imgur.com/xyfynLd.png)
    <center> <b>Figure 2</b>: Overall Optimisation Process</center>

- Usually, optimisation is done for a specific target like
    - Speed, which targets Single execution time and latency.
    - Memory Usage, which targets Maximum resident set and allocation count.
    - Throughput, which is the number of results per unit of time and number of results per resource spend.
    - Quality, which is Application and use-case dependent.

    This analysis targets Speed and Memory Usage since those are the fundamental things which should make any encoder production-ready and impact will be very high.

- There is no need to encode full videos which are having 1000 Frames to do benchmarking and per code-path analysis.
    - One thing which is to be taken care is when the encoding preset is being tuned is, it should encode more than RDO Look Ahead like around 100 Frames for fair testing and good results of different types of the clip.
    - The Complexity of an encoder also depends on how large the functions and code blocks are there, see the below table for a quick comparison between rav1e and SVT-AV1.
        <center>

        | Language         | files | blank | comment | code   |
        |------------------|-------|-------|---------|--------|
        | Assembly         | 44    | 4405  | 3371    | 71208  |
        | Rust             | 117   | 4494  | 5455    | 48370  |
        | YAML             | 6     | 49    | 3       | 669    |
        | Markdown         | 7     | 157   | 0       | 376    |
        | TOML             | 8     | 37    | 10      | 232    |
        | Bourne Shell     | 6     | 38    | 20      | 151    |
        | Python           | 3     | 39    | 17      | 144    |
        | Jupyter Notebook | 1     | 0     | 478     | 129    |
        | SUM:             | 192   | 9219  | 9354    | 121279 |

        </center>
       <center><b>Table 1</b>: Lines of Code Distrubution of rav1e</center>

        <center>

        | Language       | files | blank | comment |   code |
        |----------------|-------|-------|---------|--------|
        | Assembly       | 44    | 4419  | 3364    | 74276  |
        | C              | 81    | 2934  | 3180    | 30232  |
        | C/C\+\+ Header | 71    | 816   | 2406    | 3578   |
        | NAnt script    | 27    | 210   | 0       | 2506   |
        | YAML           | 2     | 46    | 0       | 486    |
        | JSON           | 1     | 0     | 0       | 427    |
        | Markdown       | 4     | 88    | 0       | 146    |
        | Python         | 1     | 12    | 28      | 35     |
        | SUM:           | 231   | 8506  | 8978    | 111686 |

        </center>
        <center> <b>Table 2</b>: Lines of Code Distrubution of dav1d</center>

        <center>

        | Language       | files | blank | comment | code   |
        |----------------|-------|-------|---------|--------|
        | C              | 245   | 29282 | 19765   | 229148 |
        | C/C\+\+ Header | 289   | 9646  | 16714   | 64523  |
        | C\+\+          | 149   | 8223  | 11165   | 44228  |
        | make           | 25    | 3504  | 1925    | 7009   |
        | Markdown       | 20    | 1429  | 0       | 4438   |
        | Python         | 30    | 1396  | 2616    | 4436   |
        | Assembly       | 12    | 606   | 494     | 4291   |
        | CMake          | 100   | 579   | 657     | 3618   |
        | XML            | 14    | 0     | 0       | 1510   |
        | diff           | 2     | 4     | 102     | 1124   |
        | YAML           | 8     | 31    | 26      | 708    |
        | m4             | 3     | 52    | 60      | 393    |
        | Bourne Shell   | 3     | 35    | 86      | 382    |
        | DOS Batch      | 1     | 5     | 0       | 119    |
        | NAnt script    | 1     | 7     | 0       | 47     |
        | SUM:           | 902   | 54799 | 53610   | 365974 |

        <center> <b>Table 3</b>: Lines of Code Distrubution of SVT-AV1</center>
        <br></br>
        </center>

     One thing which is to be noted is that SVT-AV1 is having both encoder and decoder while rav1e is encoder alone if the decoder is also added to rav1e, dav1d, AV1 Decoder also, still it will be having half of SVT-AV1 and 1/3 of libaom in terms of lines of code, which makes a good starting point.

- The code-coverage is another important factor, having more than 50\% code-coverage is always good to have for any software, while rav1e is having around 77\% code-coverage.

- For testing based on [IETF NETVC testing draft](https://tools.ietf.org/html/draft-ietf-netvc-testing-09),  Objective-1 Tests are being focused primarily. The Videos should be YUV 4:2:0 Y4M Uncompressed Video which does not have Audio in it for better bitrate and faster encodes.

- For having better analysis, the encoder should have a better dynamic analysis for faster research and development,  for that, the following changes are being introduced,
    - Introduction of rav1e and dav1d to RTC Video Quality tool for rapid analysis between encoders.
    - Introduction of new Library called v-frame for having Video configuration, definition and its missionary.
    - Migration of AV-Metrics, the Library which calculates video metrics like PSNR, SSIM, A-PSNR, SSIM, CIEDE2000, PSNR-HVS etc to use v-frame as standard configuration.
    - Integrate AV-Metrics to rav1e for rapid encoding analysis

    The above-mentioned things will be explained in the further sections.



## Into Depths of Analysis and its tools

### **RTC Video Quality Tool**
RTC Video Quality Tool([rtc_tool](https://github.com/vibhoothiiaanand/rtc-video-quality)) is a project started by Google which is used for quick analysis between various encoders by generating graphs based on quality metrics.

The main use-case which the tool is targetting to cover is improved rapid analysis of rav1e with other industrial encoders like VP8, VP9, libaom-av1, OpenH264 which are being used to get the clear trade-offs and hotspots between encoders.

For rapid analysis of any encoder, an equally good or better decoder is required, in AV1  [dav1d](https://code.videolan.org/dav1d)(**Dav1d** is a **AV1** **D**ecoder) is the best choice.

[AreWeCompressedYet](https://arewecompressedyet.com/) is always a better choice, the fundamental issue here is faster and rapid analysis, there would be a trade-off on more accurate results which is done on AWCY but for research and development purpose, this works better.

  ![image650c](https://i.imgur.com/5Rheycr.png)
  <center> <b>Figure 3</b>: Graphs generated using rtc_tool</center>

The figures depict the variation of different encoders by producing more than 80 different graphs using this tool which also includes frame data with varying bitrates.

Command which is used to do encoding comparison between multiple encoders including rav1e:

```bash
$ wget https://mf4.xiph.org/~ltrudeau/videos/nyan.y4m
$ ./generate_data.py --out=lib-rav1e.txt --encoders=aom-good:av1,
libvpx-rt:vp9,rav1e-default:rav1e nyan.y4m
$ python generate_graphs.py --out-dir rav1e_graphs  lib-rav1e.txt
```


### **V-Frame**
[v-frame](https://crates.io/crates/v_frame) is a library released by Team rav1e in March 2020, v-frame is library extracted from rav1e to have Video Configurations structures and it's missionary.
v-frame have moved the following to a separate library:

- Pixel: The pixel is located in `v_frame/src/pixel.rs` where there are several methods viz, Trait for casting between Primitive Types (`CastFromPrimitive`) like u8,u16, i16, i32. Types of Pixel that can be used, in v-frame is U8, used for 8-bit pixels, and U16 used for higher bitrate (HBDs) like 10 or 12 bits per pixel. The main trait of Pixel which is a type that can be used as a pixel type. v-frame's pixel is traits is similar to what the pixel is in rav1e.

- Math: The Plane abstraction is using different maths functions like the floor, the ceiling of log2, aligning with the power of 2 and also then shifting values, finding Most Significant Bits, and rounding. These are just changed location form rav1e's util to `v_frame` (`src/util/math.rs` -> `v_frame/src/math.rs`)

- Plane: Plane in `v_frame` is again similar to what is being found in rav1e. In the Plane implementation wrap function is being modified or changed to `from_slice` where the input arguments are also changed to Address of an Array of Type Pixel(`&[T]`) instead of Vector of type(`Vec<T>`) so a better representation of data which is easier refutable rather than an enclosing in a vector, and replacing the subsequent function of wrap with `from_slice`. For example:

    `Plane:::wrap(data, width)` ->
     `Plane::from_slice(&data, width)`

- Chroma Sampling: Moving the ChromaSampling to v-frame so the Frame structure can use this. There are few things which are modified for cross-compatibility. Firstly, there is serialize feature which is being used by chromaSampling API which is using [serde](https://crates.io/crates/serde) library with `derive` feature as a dependency. This is added as a feature among main Cargo definition as it is an optional dependency and sharing dependency is clean in rust.

- Frame: Moving of Frame is not a straightforward method among the v-frame as it was having a lot of dependencies, function calls. The changes should be minimal so it will be easier to adapt an external Frame Structure in upstream.

- Earlier padding constant was present inside frame implementation. Now, it is moved outside the implementation.

    `const LUMA_PADDING: usize = SB_SIZE + FRAME_MARGIN;` ([L23](https://github.com/xiph/rav1e/blob/master/src/frame/mod.rs#L22))

- Luma padding is required for new frame allocation, while it is not available in the initialisation of new frame outside rav1e. Now it is required to initialise frames with padding constant as an additional argument for calculation of chroma padding.

- In favour of minimal changes to upstream, a new trait  FrameAlloc for Frame Structure is defined in upstream(rav1e), which defines new function without padding constant and returns the value.

    ```rust
    {
    /// Public Trait Interface for Frame Allocation
     pub trait FrameAlloc {
       /// Initialise new frame default type
       fn new(width: usize, height: usize,
       chrop_sampling: ChromaSampling) -> Self;
     }
     impl<T: Pixel> FrameAlloc for Frame<T> {
       /// Creates a new frame with the given parameters.
       /// new function calls new_with_padding function which takes
       ///  luma_padding as parameter
       fn new(
         width: usize, height: usize, chroma_sampling: ChromaSampling,
       ) -> Self {
         v_frame::frame::Frame::new_with_padding(
           width, height, chroma_sampling, LUMA_PADDING )
       }
     }
     ```
     For more details check [src/frame/mod.rs](https://github.com/xiph/rav1e/blob/master/src/frame/mod.rs#L46)

Likewise, there are new simple trait functions for Calculating Padding ([FramePad](https://github.com/xiph/rav1e/blob/master/src/frame/mod.rs#L69)), for the new tile of a frame([AsTile](https://github.com/xiph/rav1e/blob/master/src/frame/mod.rs#L87)) which has functions for tiles and mutable tiles(as\_tile, as\_tile\_mut) for Frame because these functions are written inside Frame implementation. These are encoder specific ones which can be isolated from the v-frame.

### AV-Metrics

[AV-Metrics](https://github.com/rust-av/av-metric) is a library introduced by Rust Audio-Video([Rust-AV](https://github.com/rust-av)) organisation which is targeted to measure the quality of compressed video comparing with uncompressed video and gives an output which consists of

- PSNR: Peak Signal to Noise Ratio or formally known as PSNR, is a traditional signal quality metrics measured in decibels. It is directly derived from mean square error(MSE) or its square root (RMSE). The formula being used to calculate is \
    `20 * log10 { (MAX / RSE)}`\
    or\
    `10 * log10 ( MAX^2 / MSE )`

    This metric is being applied to both Luma(Y) and Chroma Planes (Cb/Cr). Also to be added that the error is computed over all the pixels in the video.

- APSNR: Aligned PSNR is a metrics which is used to improve the accuracy of conventional PSNR. In APSNR there defines a dynamic window size as\
    `w = sumFL + 1`\
    Where,\
    `w` = window size,\
    `sumFL` = total of frame loss,\
    APSNR and PSNR are related but it is not possible to calculate full-video APSNR from whole-video PSNR, APSNR comes from a different method of averaging together the per-frame PSNRs.

- PSNR-HVS: The Peak Signal-to-Noise Ratio for the Human Visual System (PSNR-HVS) metric performs a DCT(Discrete Cosine Transformation) of 8x8 blocks of the image, weights the coefficient and calculates the PSNR of those coefficients. In AV-Metrics, weights are taken by Daala Tools. The normalized inverse quantisation matrix for 8x8 DCT is not the JPEG based matrix and better correlation to Mean Observer Score(MOS) than PSNR.

- SSIM: Structural Similarity Image Metrics([SSIM](http://www.cns.nyu.edu/pub/eero/wang03-reprint.pdf)) is a still image quality metric introduced which computes a score for each individual pixel, using a window of neighbouring pixels after which the scores are averaged to produce a global score for the entire image. Original Paper produces a score between 0 and 1. In other words, the measurement or prediction of image quality is based on an initial uncompressed or distortion-free image as a reference. SSIM is designed to improve over traditional methods such as PSNR, MSE. For making it more linear in accordance with BD-Rate computation for videos.\
    `10 * log10(1-SSIM)`

- MSSIM: Multi-Scale SSIM([MSSIM](http://www.cns.nyu.edu/~zwang/files/papers/msssim.pdf) is a variant of SSIM computed over subsampled versions of an image. It is designed to be a more accurate metric than SSIM. The multi-Scale method is a convenient way to incorporate image details at different resolutions. Inputs are a reference and distorted image signals, the system iteratively applies a low-pass filter and downsamples the filtered image by a factor of 2.

- CIEDE2000: [CIEDE200](http://dx.doi.org/10.1155/2012/273723) is a metric based on CIEDE color distances. It generates a single score taking all three(Y, Cb, Cr) Chroma Planes. Like other metrics, CIEDE200 does not consider Structural similarity. Implementation of CIEDE2000 in AV-Metrics includes lookup tables, binomial series and multiple conversions of different colour metrics.

These are very much required for Objective Video Quality Assessment. Objective Assessments are done in place of subjective Video Quality Assessment for easy and iterative experiments. Most of the above metrics apply to luma planes, and individually to planes in frames.

### Integration of v-frame in AV-Metrics

[Migrating](https://github.com/rust-av/av-metrics/pull/67) to v-frame type in AV-Metrics is very much required because common definitions and structures so it can be integrated easily into encoders and will be faster.

- Introduction of v-frame as a crate: The v-frame is being added as a dependency in the Cargo.toml of the AV-Metrics.

- Using v-frame's Plane instead of PlaneData: Earlier the FrameInfo Structure had planes of type PlaneData which is having a width, height and data of type Vec as elements. But all the data are present in the Plane struct itself which is found in v-frame. The PlaneConfig from Plane Structure for getting the video Width, Height will be used.

- Improvising preliminary checks of input video: Earlier the PlaneData Implementation has checks for width, the height of both input videos to find any mismatch, now the EncoderConfig from v-frame can be used to validate both, thus giving strict check because PlaneConfig has more than width and height like Data stride, Allocated height in Pixels, Width, Height, Decimator along the X and Y axis, Number of padding pixels on right and bottom, and also X and y where the data starts. This is made plausible by having a trait PlaneCompare for Plane.

- Addition of ChromaWeight Trait: The chroma weight function from chroma sampling implementation is introduced as a trait. The weight is defined as follows, these are nothing but the relative impact of the chroma planes compared to luma.
     ```rust
     ChromaSampling::Cs420 => 0.25
     ChromaSampling::Cs422 => 0.5
     ChromaSampling::Cs444 => 1.0
     ChromaSampling::Cs400 => 0.0
     ```

- Using Pixel from v-frame

- Having Stride for calculation: Currently, the assertion in the calculation of PSNR-HVS checks plane's length "=" with the product of width and height which is replaced with ">=" product of stride value and height, so more strict check gives better results. During the initial tests of integration to encode, it was noticed that without this strict, encoder crashes and panics.
     ```diff
     - assert!(plane1.data.len() == width * height);
     + assert!(plane1.data.len() >= stride * height);
     ```

#### Using v-frame for decoding:

   - The new VideoDetails structure is used to store details of video like width, height, bit-depth, Chroma Sampling and position of the video, timebase of a video and also padding constant.

   -  Having a Rational structure which helps to create new rational number, return as reciprocal and also return as a floating-point number and also being used by Time Base. Calling `get_video_details` function which is of VideoDetails Structs as return type will give Video Details.

   -  The `read_video_frame` function which reds the next frame from input video now as VideoDetails as an additional argument. The Decoder binary needs to be refactored in such a way that the Frame, Plane are used from the v-frame.

   - The function which returns the chroma sampling which is being matched, now it is same only but the input is of the type `y4m::ColorSpace` and return type are `(ChromaSamplng, ChromaSamplingPosition)`.

   - The `get_video_details` function is being made for `y4`::Decoder<'_', R>`.

   -  Most of the values are being returned with the help of constructors and these values are being returned in this function.

   -  Updating Converting Chroma Data: Currently, the chroma data function takes PlaneData and chroma position, and bit depth as input with return type as `PlaneData<T>`. The additional structure Plane Data is obsolete and now it is being directly used from Plane. The plane data is now a simple clone of Plane's Data while earlier it was more complex enclosed in a Vector and having the length of the data and returns Plane Structure which has both data and PlaneConfig's Clone.

   -  Updating Converting Chroma Data: Currently, the chroma data function takes PlaneData and chroma position, and bit depth as input with return type as `PlaneData<T>`. The additional structure Plane Data is obsolete and now it is being directly used from Plane. The plane data is now a simple clone of Plane's Data while earlier it was more complex enclosed in a Vector and having the length of the data and returns Plane Structure which has both data and PlaneConfig's Clone.
        ```diff
        - let mut output_data = vec![T::cast_from(0u8);plane_data.data.len()];
        + let mut output_data = plane_data.data.clone();
        ```
      The Algorithms is ported from daala-tools, the vertical chroma sample position must be realigned to get accurate results else it will not get, having precise chroma calculation in an encoder point of view does not matter much but for a library, it does matter.

      In future, the input as y4m frame and having chroma width and bytes as parameters and returns newly constructed frame.
      ```rust
      convert_chroma_data(frame.get_u_plane(), chroma_s_pos, bit_depth, chroma_width, bytes);
      ```


   - The function `read_video_frame` function reads Frames and returns is of the type FrameInfo enclosed in Result.
        ```rust
        fn read_video_frame<T: Pixel>(&mut self,cfg: &VideoDetails) -> Result<FrameInfo<T>, ()>
        ```
     Earlier the function was returning FrameInfo Structure, which is now obsolete and replaced by a new frame initialised inside the method. For U, V Planes, conversion of chroma data is also requried. The `read_frame` was having FrameInfo piped to frame which is not required because passing information directly by doing calculation inside map function is faster which also includes the calculation of planes from y4m decoder and converting of chroma data happens. In the end, these planes are returned adding this to Frame Info.

Overall with the new y4m decode approach, decoding capability was improved and it is now more efficient.


### Integration of AV-Metrics in rav1e
Once both v-frame and av-metrics are built with the same Frame, Plane, PlaneConfig, Pixel, ChromaSampling, [integration](https://github.com/xiph/rav1e/pull/2184) of  AV-Metrics into rav1e without touching rav1e API Internals will be better. For that the following things are being done:

- Introduction of Reference Frame for Packet API: For the calculation of any metrics or screen change, the source/input/[reference frame](https://github.com/xiph/rav1e/pull/2186) is required which are taken from the  FrameData(input) to packets which is of the type `Option<Arc<Frame<T>>>`

- Breaking API: The `show_psnr` from EncoderConfig structure and psnr from Packet Structure are not useful thus removing because these are obsolete and are not required for calculation of metrics.

- Introduction of `--metrics`: The [common.rs](https://github.com/xiph/rav1e/blob/master/src/bin/common.rs)(`src/bin/common.rs`) defines CLI Options are being written. Also, the calculation of PSNR is also covered easily with this, it is happening like this:

    ```rust
    let metrics_enabled = if matches.is_present("METRICS") {
         MetricsEnabled::All
    } else if matches.is_present("PSNR") {
         MetricsEnabled::Psnr
    } else {
         MetricsEnabled::None
    };
    ```
    Also to be noted that `metrics_enabled` is being added to CliOptions Structure.

- Calculation of QualityMetrics: With the new QualityMetrics struct, which has PSNR, PSNR-HVS, SSIM, MS-SSIM, CIEDE2000, A-PSNR(In-Future), VMAF(In-Future).The `calculate_frame_metrics` method takes two frames(source and output), bit depth, chroma sampling, metrics enabled as inputs and has a return type of QualityMetrics where it returns after calculation.

- Calculation of metrics happens when the packet is being received in the main [encode loop](https://github.com/xiph/rav1e/blob/master/src/bin/rav1e.rs)(`src/bin/rav1e.rs`) based on the input and output frame from Packet.

- In the stats.rs itself, functions which fetches metrics from QualityMetrics strut and displays it to users.\
    ```rust
    let psnr_hvs = sum_metric(&self.frame_info, |fi| fi.metrics.psnr_hvs.unwrap().avg);
    ```
- In production, the encoder will be independent of av-metrics and do not need as a dependency, The [rav1e library](https://github.com/xiph/rav1e/blob/master/src/lib.rs)(`src/lib.rs`) will be not modified and there will no metrics module which will inside the binary and will be handled using config(cfg) flags.
     ```rust
     #[cfg(feature = "metrics")]
     ```

## How it looks now!!
 ![image650c](https://i.imgur.com/489dqFB.png)

<center> <b>Figure 4</b>:CLI Output</center>


## Thanks
I would like to thank to Luca Barbato([lu_zero](https://github.com/lu-zero)), Josh Holmer([soichiro](https://github.com/shssoichiro)), David Micheal Barr([barrbrain](http://ba.rr-dav.id.au)), Nathan Egge([unlord](https://developer.mozilla.com/communities/people/nathan-egge/)), Christopher Montgomery([xiphmont](https://people.xiph.org/~xiphmont/demo)) and also all others from Team rav1e for the continous help and debugging, reviewing of the whole research work.

<br></br>

Freely,\
~mindfreeze