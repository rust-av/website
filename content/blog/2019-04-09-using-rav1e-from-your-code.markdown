+++
date = "2019-04-09"
title = "Using rav1e - from your own code"
description = "Luca Barbato"
menu = "blog"
weight = 998
+++

# AV1, Rav1e, Crav1e, an intro

> [AV1](https://aomedia.org/av1-features/) is a modern video codec brought to you by an alliance of [many](https://aomedia.org/membership/members/) different bigger and smaller players in the multimedia field.
> I'm part of the [VideoLAN](https://videolan.org) organization and I spent quite a bit of time on this codec lately.

---
> [rav1e](https://github.com/xiph/rav1e/): The safest and fastest AV1 encoder, built by many volunteers and Mozilla/Xiph developers.
> It is written in [rust](https://rustlang.org) and strives to provide good speed, quality and stay maintainable.

---
> [crav1e](https://github.com/lu-zero/crav1e): A companion [crate](https://doc.rust-lang.org/book/crates-and-modules.html), written by [yours truly](https://github.com/lu-zero), that provides a C-API, so the encoder can be used by C libraries and programs.

---
This article will just give a quick overview of the API available right now and it is mainly to help people start using it and hopefully report issues and problem.

## Rav1e API
The current API is built around the following 4 structs and 1 enum:

- `struct Frame`: The raw pixel data
- `struct Packet`: The encoded bitstream
- `struct Config`: The encoder configuration
- `struct Context`: The encoder state

---

- `enum EncoderStatus`: Fatal and non-fatal condition returned by the `Context`methods.

### Config
The `Config` struct currently is simply constructed.
``` rust
    struct Config {
        enc: EncoderConfig,
        threads: usize,
    }
```
The `EncoderConfig` stores all the settings that have an impact to the actual bitstream while settings such as `threads` are kept outside.
``` rust
    let mut enc = EncoderConfig::with_speed_preset(speed);
    enc.width = w;
    enc.height = h;
    enc.bit_depth = 8;
    let cfg = Config { enc, threads: 0 };
```
> **NOTE**: Some of the fields above may be shuffled around until the API is marked as stable.

#### Methods
##### new_context
``` rust
    let cfg = Config { enc, threads: 0 };
    let ctx: Context<u8> = cfg.new_context();
```
It produces a new encoding context. Where `bit_depth` is *8*, it is possible to use an optimized `u8` codepath, otherwise `u16` must be used.

### Context
It is produced by `Config::new_context`, its implementation details are hidden.

#### Methods
The `Context` can be grouped into **essential**, **optional** and **convenience**.
``` rust
    // Essential API
    pub fn send_frame<F>(&mut self, frame: F) -> Result<(), EncoderStatus>
      where F: Into<Option<Arc<Frame<T>>>>, T: Pixel;
    pub fn receive_packet(&mut self) -> Result<Packet<T>, EncoderStatus>;
```
The encoder works by processing each `Frame` fed through `send_frame` and producing each `Packet` that can be retrieved by `receive_packet`.

``` rust
    // Optional API
    pub fn container_sequence_header(&mut self) -> Vec<u8>;
    pub fn get_first_pass_data(&self) -> &FirstPassData;
```
Depending on the container format, the `AV1`  Sequence Header could be stored in the extradata. `container_sequence_header` produces the data pre-formatted to be simply stored in `mkv` or  `mp4`.

`rav1e` supports multi-pass encoding and the encoding data from the first pass can be retrieved by calling `get_first_pass_data`.

``` rust
    // Convenience shortcuts
    pub fn new_frame(&self) -> Arc<Frame<T>>;
    pub fn set_limit(&mut self, limit: u64);
    pub fn flush(&mut self) {
```

- `new_frame()` produces a frame according to the dimension and pixel format information in the Context.
- `flush()` is functionally equivalent to call `send_frame(None)`.
- `set_limit()`is functionally equivalent to call `flush()`once `limit` frames are sent to the encoder.

### Workflow
The workflow is the following:

1. Setup:
    - Prepare a `Config`
    - Call `new_context` from the `Config` to produce a `Context`

2. Encode loop:
    - Pull each `Packet` using `receive_packet`.
    - If `receive_packet` returns `EncoderStatus::NeedMoreData`
        - Feed each `Frame` to the `Context` using `send_frame`

3. Complete the encoding
    - Issue a `flush()` to encode each pending `Frame` in a final `Packet`.
    - Call `receive_packet` until `EncoderStatus::LimitReached` is returned.


## Crav1e API
The [crav1e](https://github.com/lu-zero/crav1e) API provides the same structures and features beside few key differences:

- The `Frame`, `Config`,  and `Context` structs are *opaque*.
``` c
typedef struct RaConfig RaConfig;
typedef struct RaContext RaContext;
typedef struct RaFrame RaFrame;
```

- The `Packet` struct exposed is much simpler than the `rav1e` original.
``` c
typedef struct {
    const uint8_t *data;
    size_t len;
    uint64_t number;
    RaFrameType frame_type;
} RaPacket;
```

- The EncoderStatus includes a `Success` condition.
``` c
typedef enum {
    RA_ENCODER_STATUS_SUCCESS = 0,
    RA_ENCODER_STATUS_NEED_MORE_DATA,
    RA_ENCODER_STATUS_ENOUGH_DATA,
    RA_ENCODER_STATUS_LIMIT_REACHED,
    RA_ENCODER_STATUS_FAILURE = -1,
} RaEncoderStatus;
```

### RaConfig
Since the configuration is `opaque` there are a number of functions to assemble it:

- `rav1e_config_default` allocates a default configuration.
- `rav1e_config_parse` and `rav1e_config_parse_int` set a specific `value` for a specific field selected by a `key` string.
- `rav1e_config_set_${field}` are specialized setters for complex information such as the color description.

``` c
RaConfig *rav1e_config_default(void);

/**
 * Set a configuration parameter using its key and value as string.
 * Available keys and values
 * - "quantizer": 0-255, default 100
 * - "speed": 0-10, default 3
 * - "tune": "psnr"-"psychovisual", default "psnr"
 * Return a negative value on error or 0.
 */
int rav1e_config_parse(RaConfig *cfg, const char *key, const char *value);

/**
 * Set a configuration parameter using its key and value as integer.
 * Available keys and values are the same as rav1e_config_parse()
 * Return a negative value on error or 0.
 */
int rav1e_config_parse_int(RaConfig *cfg, const char *key, int value);

/**
 * Set color properties of the stream.
 * Supported values are defined by the enum types
 * RaMatrixCoefficients, RaColorPrimaries, and RaTransferCharacteristics
 * respectively.
 * Return a negative value on error or 0.
 */
int rav1e_config_set_color_description(RaConfig *cfg,
                                       RaMatrixCoefficients matrix,
                                       RaColorPrimaries primaries,
                                       RaTransferCharacteristics transfer);

/**
 * Set the content light level information for HDR10 streams.
 * Return a negative value on error or 0.
 */
int rav1e_config_set_content_light(RaConfig *cfg,
                                   uint16_t max_content_light_level,
                                   uint16_t max_frame_average_light_level);

/**
 * Set the mastering display information for HDR10 streams.
 * primaries and white_point arguments are RaPoint, containing 0.16 fixed point values.
 * max_luminance is a 24.8 fixed point value.
 * min_luminance is a 18.14 fixed point value.
 * Returns a negative value on error or 0.
 */
int rav1e_config_set_mastering_display(RaConfig *cfg,
                                       RaPoint primaries[3],
                                       RaPoint white_point,
                                       uint32_t max_luminance,
                                       uint32_t min_luminance);

void rav1e_config_unref(RaConfig *cfg);
```
The bare minimum setup code is the following:
``` c
    int ret = -1;
    RaConfig *rac = rav1e_config_default();
    if (!rac) {
        printf("Unable to initialize\n");
        goto clean;
    }

    ret = rav1e_config_parse_int(rac, "width", 64);
    if (ret < 0) {
        printf("Unable to configure width\n");
        goto clean;
    }

    ret = rav1e_config_parse_int(rac, "height", 96);
    if (ret < 0) {
        printf("Unable to configure height\n");
        goto clean;
    }

    ret = rav1e_config_parse_int(rac, "speed", 9);
    if (ret < 0) {
        printf("Unable to configure speed\n");
        goto clean;
    }
```
### RaContext
As per the `rav1e` API, the context structure is produced from a configuration and the same *send-receive* model is used.
The convenience methods aren't exposed and the frame allocation function is actually essential.

``` c
// Essential API
RaContext *rav1e_context_new(const RaConfig *cfg);
void rav1e_context_unref(RaContext *ctx);

RaEncoderStatus rav1e_send_frame(RaContext *ctx, const RaFrame *frame);
RaEncoderStatus rav1e_receive_packet(RaContext *ctx, RaPacket **pkt);
```
``` c
// Optional API
uint8_t *rav1e_container_sequence_header(RaContext *ctx, size_t *buf_size);
void rav1e_container_sequence_header_unref(uint8_t *sequence);
```
### RaFrame
Since the frame structure is opaque in C, we have the following functions to create, fill and dispose of the frames.

``` c
RaFrame *rav1e_frame_new(const RaContext *ctx);
void rav1e_frame_unref(RaFrame *frame);

/**
 * Fill a frame plane
 * Currently the frame contains 3 planes, the first is luminance followed by
 * chrominance.
 * The data is copied and this function has to be called for each plane.
 * frame: A frame provided by rav1e_frame_new()
 * plane: The index of the plane starting from 0
 * data: The data to be copied
 * data_len: Lenght of the buffer
 * stride: Plane line in bytes, including padding
 * bytewidth: Number of bytes per component, either 1 or 2
 */
void rav1e_frame_fill_plane(RaFrame *frame,
                            int plane,
                            const uint8_t *data,
                            size_t data_len,
                            ptrdiff_t stride,
                            int bytewidth);
```


### RaEncoderStatus
The encoder status enum is returned by the `rav1e_send_frame` and `rav1e_receive_packet` and it is possible already to arbitrarily query the context for its status.
``` c
RaEncoderStatus rav1e_last_status(const RaContext *ctx);
```

To simulate the rust [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) functionality a `to_str` function is provided.
``` c
char *rav1e_status_to_str(RaEncoderStatus status);
```

### Workflow
The *C* API workflow is similar to the *Rust* one, albeit a little more verbose due to the error checking.
``` c
    RaContext *rax = rav1e_context_new(rac);
    if (!rax) {
        printf("Unable to allocate a new context\n");
        goto clean;
    }
```
``` c
    RaFrame *f = rav1e_frame_new(rax);
    if (!f) {
        printf("Unable to allocate a new frame\n");
        goto clean;
    }
```
``` c
while (keep_going(i)){
     RaPacket *p;
     ret = rav1e_receive_packet(rax, &p);
     if (ret < 0) {
         printf("Unable to receive packet %d\n", i);
         goto clean;
     } else if (ret == RA_ENCODER_STATUS_SUCCESS) {
         printf("Packet %"PRIu64"\n", p->number);
         do_something_with(p);
         rav1e_packet_unref(p);
         i++;
     } else if (ret == RA_ENCODER_STATUS_NEED_MORE_DATA) {
         RaFrame *f = get_frame_by_some_mean(rax);
         ret = rav1e_send_frame(rax, f);
         if (ret < 0) {
            printf("Unable to send frame %d\n", i);
            goto clean;
        } else if (ret > 0) {
        // Cannot happen in normal conditions
            printf("Unable to append frame %d to the internal queue\n", i);
            abort();
        }
     } else if (ret == RA_ENCODER_STATUS_LIMIT_REACHED) {
         printf("Limit reached\n");
         break;
     }
}
```

## In closing
This article was mainly a good excuse to try [dev.to](https://dev.to) and see write down some notes and clarify my ideas on what had been done API-wise so far and what I should change and improve.

If you managed to read till here, your feedback is really welcome, please feel free to comment, try the software and open issues for [crav1e](https://github.com/lu-zero/crav1e/issues/new) and [rav1e](https://github.com/xiph/rav1e/issues/new).

### Coming next

- Working **crav1e** got me to see what's good and what is lacking in the [c-interoperability story](https://blogs.gentoo.org/lu_zero/2018/12/30/making-and-using-c-compatible-libraries-in-rust-present-and-future/) of *rust*, now that [this](https://github.com/rust-lang/cargo/pull/6298) landed I can start crafting and publishing better tools for it and maybe I'll talk more about it here.
- Soon **rav1e** will get more threading-oriented features, some benchmarking experiments will happen soon.

### Thanks
- Special thanks to [Derek](https://github.com/dwbuiten) and [Vittorio](https://github.com/kodabb/) spent lots of time integrating `crav1e` in larger software and gave precious feedback in what was missing and broken in the initial iterations.
- Thanks to [David](http://github.com/barrbrain/) for the review and editorial work.
- Also thanks to [Matteo](https://dev.to/naufraghi) for introducing me to [dev.to](https://dev.to).