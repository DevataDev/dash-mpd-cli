# dash-mpd-cli

A commandline application for downloading media content from a DASH MPD file, as used for on-demand
replay of TV content and video streaming services like YouTube.

[![Crates.io](https://img.shields.io/crates/v/dash-mpd-cli)](https://crates.io/crates/dash-mpd-cli)
[![Released API docs](https://docs.rs/dash-mpd-cli/badge.svg)](https://docs.rs/dash-mpd-cli/)
[![CI](https://github.com/emarsden/dash-mpd-cli/workflows/build/badge.svg)](https://github.com/emarsden/dash-mpd-cli/actions/)
[![Dependency status](https://deps.rs/repo/github/emarsden/dash-mpd-cli/status.svg)](https://deps.rs/repo/github/emarsden/dash-mpd-cli)
[![LICENSE](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE-MIT)

![Terminal capture](https://risk-engineering.org/emarsden/dash-mpd-cli/terminal-capture.svg)


[DASH](https://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP) (dynamic adaptive
streaming over HTTP), also called MPEG-DASH, is a technology used for media streaming over the web,
commonly used for video on demand (VOD) services. The Media Presentation Description (MPD) is a
description of the resources (manifest or “playlist”) forming a streaming service, that a DASH
client uses to determine which assets to request in order to perform adaptive streaming of the
content. DASH MPD manifests can be used with content encoded in different formats and containers,
including H264/MP4, HEVC/MP4 and VP9/WebM. There is a good explanation of adaptive bitrate video
streaming at [howvideo.works](https://howvideo.works/#dash).

This commandline application allows you to download content (audio or video) described by an MPD
manifest. This involves selecting the alternative with the most appropriate encoding (in terms of
bitrate, codec, etc.), fetching segments of the content using HTTP or HTTPS requests and muxing
audio and video segments together. There is also some preliminary support for downloading subtitles
(mostly WebVTT, TTML and SMIL formats, with some support for wvtt format). 

This application builds on the [dash-mpd](https://crates.io/crates/dash-mpd) crate.


## Installation

**Binary releases** are [available on GitHub](https://github.com/emarsden/dash-mpd-cli/releases) for
GNU/Linux on AMD64 (statically linked against musl libc to avoid glibc versioning problems), Microsoft
Windows on AMD64 and MacOS on AMD64. These are built automatically on the GitHub continuous
integration infrastructure.

You can also **build from source** using an [installed Rust development
environment](https://www.rust-lang.org/tools/install):

```shell
cargo install dash-mpd-cli
```

You should also install the following **dependencies**:

- the mkvmerge commandline utility from the [MkvToolnix](https://mkvtoolnix.download/) suite, if you
  download to the Matroska container format (`.mkv` filename extension). mkvmerge is used as a
  subprocess for muxing (combining) audio and video streams. See the `--mkvmerge-location`
  commandline argument if it's not installed in a standard location.

- [ffmpeg](https://ffmpeg.org/) or [vlc](https://www.videolan.org/vlc/) to download to the MP4
  container format, also for muxing audio and video streams (see the `--ffmpeg-location` and
  `--vlc-location` commandline arguments if these are installed in non-standard locations).

- the MP4Box commandline utility from the [GPAC](https://gpac.wp.imt.fr/) project, if you want to
  test the preliminary support for retrieving subtitles in wvtt format. If it's installed, MP4Box
  will be used to convert the wvtt stream to more widely recognized SRT format.


This crate is tested on the following **platforms**:

- Linux on AMD64 (x86-64) and Aarch64 architectures

- MacOS on AMD64 and Aarch64 architectures

- Microsoft Windows 10 on AMD64

- Android 12 on Aarch64 via [termux](https://termux.dev/) (you'll need to install the rust, binutils
  and ffmpeg packages)

- OpenBSD on AMD64 (occasionally)



## Usage

```
Download content from an MPEG-DASH streaming media manifest

Usage: dash-mpd-cli [OPTIONS] <MPD-URL>

Arguments:
  <MPD-URL>  URL of the DASH manifest to retrieve

Options:
      --user-agent <user-agent>
          
      --proxy <URL>
          URL of Socks or HTTP proxy (eg. https://example.net/ or socks5://example.net/)
      --timeout <SECONDS>
          Timeout for network requests (from the start to the end of the request), in seconds
      --sleep-requests <SECONDS>
          Number of seconds to sleep between network requests (default 0)
      --source-address <source-address>
          Source IP address to use for network requests, either IPv4 or IPv6. Network requests will be made using the version of this IP address (eg. using an IPv6 source-address will select IPv6 network traffic).
      --quality <quality>
          Prefer best quality (and highest bandwidth) representation, or lowest quality [possible values: best, worst]
      --prefer-language <LANG>
          Preferred language when multiple audio streams with different languages are available. Must be in RFC 5646 format (eg. fr or en-AU). If a preference is not specified and multiple audio streams are present, the first one listed in the DASH manifest will be downloaded.
      --video-only
          If the media stream has separate audio and video streams, only download the video stream
      --audio-only
          If the media stream has separate audio and video streams, only download the audio stream
      --write-subs
          Write subtitle file, if subtitles are available
      --keep-video
          Don't delete the file containing video once muxing is complete.
      --keep-audio
          Don't delete the file containing audio once muxing is complete.
      --ignore-content-type
          Don't check the content-type of media fragments (may be required for some poorly configured servers)
      --add-header <NAME:VALUE>
          Add a custom HTTP header and its value, separated by a colon ':'. You can use this option multiple times.
  -q, --quiet
          
  -v, --verbose...
          Level of verbosity (can be used several times)
      --no-progress
          Disable the progress bar
      --no-xattr
          Don't record metainformation as extended attributes in the output file
      --ffmpeg-location <PATH>
          Path to the ffmpeg binary (necessary if not located in your PATH)
      --vlc-location <PATH>
          Path to the VLC binary (necessary if not located in your PATH)
      --mkvmerge-location <PATH>
          Path to the mkvmerge binary (necessary if not located in your PATH)
  -o, --output <PATH>
          Save media content to this file
  -h, --help
          Print help
  -V, --version
          Print version
```


If your filesystem supports **extended attributes**, the application will save the following
metainformation in the output file:

- `user.xdg.origin.url`: the URL of the MPD manifest
- `user.dublincore.title`: the title, if specified in the manifest metainformation
- `user.dublincore.source`: the source, if specified in the manifest metainformation
- `user.dublincore.rights`: copyright information, if specified in the manifest metainformation

You can examine these attributes using `xattr -l` (you may need to install your distribution's
`xattr` package). Disable this feature using the `--no-xattr` commandline argument.


## Muxing

The underlying library `dash-mpd-rs` has two methods for muxing audio and video streams together. If
the library feature `libav` is enabled (which is not the default configuration), muxing support is
provided by ffmpeg’s libav library, via the `ac_ffmpeg` crate. 
Otherwise, muxing is implemented by calling an external muxer, mkvmerge (from the
[MkvToolnix](https://mkvtoolnix.download/) suite), [ffmpeg](https://ffmpeg.org/) or
[vlc](https://www.videolan.org/vlc/) as a subprocess. Note that these commandline applications
implement a number of checks and workarounds to fix invalid input streams that tend to exist in the
wild. Some of these workarounds are implemented here when using libav as a library, but not all of
them, so download support tends to be more robust with the default configuration (using an external
application as a subprocess). The `libav` feature currently only works on Linux. 

The choice of external muxer depends on the filename extension of the path supplied to `--output`
or `-o` (which will be ".mp4" if you don't specify the output path explicitly):

- .mkv: call mkvmerge first, then if that fails call ffmpeg
- .mp4: call ffmpeg first, then if that fails call vlc
- other: try ffmpeg, which supports many container formats


## DASH features supported

- VOD (static) stream manifests
- Multi-period content
- XLink elements (only with actuate=onLoad semantics), including resolve-to-zero
- All forms of segment index info: SegmentBase@indexRange, SegmentTimeline,
  SegmentTemplate@duration, SegmentTemplate@index, SegmentList
- Media containers of types supported by mkvmerge, ffmpeg or VLC (this includes ISO-BMFF / CMAF / MP4, WebM, MPEG-2 TS)
- Subtitles: preliminary download support for WebVTT, TTML and SMIL streams, as well as some support for
  the wvtt format.


## Limitations / unsupported features

- Can't download from dynamic MPD manifests, that are used for live streaming/OTT TV
- Encrypted content using DRM such as Encrypted Media Extensions (EME) and Media Source Extension (MSE)
- XLink with actuate=onRequest



## License

This project is licensed under the MIT license. For more information, see the `LICENSE-MIT` file.



## Similar tools

Similar commandline tools that are able to download content from a DASH manifest:

- `yt-dlp <MPD-URL>`

- `streamlink -o /tmp/output.mp4 <MPD-URL> worst`

- `ffmpeg -i <MPD-URL> -vcodec copy /tmp/output.mp4`

- `vlc <MPD-URL>`

- `gst-launch-1.0 playbin uri=<MPD-URL>`

This application is able to download content from certain streams that do not work with other
applications (for example xHE-AAC streams which are currently unsupported by ffmpeg, streamlink, VLC,
gstreamer).


## Building

```
$ git clone https://github.com/emarsden/dash-mpd-cli
$ cd dash-mpd-cli
$ cargo build --release
$ target/release/dash-mpd-cli --help
```

The application can also be built statically with the musl-libc target on Linux. First install the
[MUSL C standard library](https://musl.libc.org/) on your system. Add linux-musl as a target to your
Rust toolchain, then rebuild for the relevant target:

```
$ sudo apt install musl-dev
$ rustup target add x86_64-unknown-linux-musl
$ cargo build --release --target x86_64-unknown-linux-musl
```

Static musl-libc builds don’t work with OpenSSL, which is why we disable default features on the
dash-mpd crate and build it with rustls-tls support.
