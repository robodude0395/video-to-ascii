<div align=center>

  ![Logo](./images/logo.svg)

<p>

  It's a simple python package to play videos in a terminal using [ASCII](https://en.wikipedia.org/wiki/ASCII) characters.

  [![Financial Contributors on Open Collective](https://opencollective.com/video-to-ascii/all/badge.svg?label=financial+contributors)](https://opencollective.com/video-to-ascii) [![PyPI version](https://badge.fury.io/py/video-to-ascii.svg)](https://badge.fury.io/py/video-to-ascii)
  [![Maintainability](https://api.codeclimate.com/v1/badges/3108b26a0bcfffd4b4fe/maintainability)](https://codeclimate.com/github/joelibaceta/video-to-ascii/maintainability)
  [![contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat)](https://github.com/joelibaceta/video-to-ascii)

</p>

![Screenshot](./images/Simpsons.apng)

</div>

<details><summary><b>Translations</b></summary>
<p>

- [🇺🇸 English](./README.md)
- [🇪🇸 Español](./translations/README_es.md)
- [🇹🇼 繁體中文](./translations/README_zh-TW.md)

<p>
</details>

## Requirements

- Python3
- PortAudio (_Only required for installation with audio support_)
- FFmpeg (_Only required for installation with audio support_)
- Linux or MacOS ... for now

## Installation

### From PyPI (Recommended)

Standard installation (without audio support):

```bash
$ pip install video-to-ascii
```

With audio support:

```bash
$ pip install video-to-ascii[audio]
```

Complete installation (with all optional dependencies):

```bash
$ pip install video-to-ascii[all]
```

### 🔄 Migration Guide

If you were using the old installation method with `--install-option`:

```bash
# ❌ Old way (no longer supported)
$ pip install video-to-ascii --install-option="--with-audio"

# ✅ New way  
$ pip install video-to-ascii[audio]
```

**Note**: The `--install-option` flag has been [deprecated by pip](https://pip.pypa.io/en/stable/news/#deprecation-of-install-option-and-global-option) and removed in pip 23.1+.

### From Source

Clone the repository and install:

```bash
$ git clone https://github.com/joelibaceta/video-to-ascii.git
$ cd video-to-ascii
$ pip install .
```

Or with audio support:

```bash
$ pip install .[audio]
```

## How to use

Just run `video-to-ascii` in your terminal

```bash
$ video-to-ascii -f myvideo.mp4
```

### Options

**`--strategy`**
Allow to choose a strategy to render the output.

![Render Strategies](./images/Strategies.png)

**`-o --output`**
Export the rendering output to a bash file to share with someone.

![Exporting](./images/export.png)

**`-a --with-audio`**
If an installation with audio support was made, you can use this option to play the audio track while rendering the video ascii characters.

## How it works

Every video is composed by a set of frames that are played at a certain frame rate.

![Video Frames](./images/imgVideoFrames.png)

Since a terminal has a specific number of rows and columns, we have to resize our video to adjust to the terminal size limitations.

![Terminal](./images/imgTerminal.png)

To reach a correct visualization of an entire frame we need to adjust the _frame height_ to match the _terminal rows_, avoiding using more _characters_ than the number of _terminal columns_.

![Resizing](./images/imgResizing.png)

When picking a character to represent a pixel we need to measure the relevance of that pixel's color in the frame, based on that we can then select the most appropriate character based on the [relative luminance](https://en.wikipedia.org/wiki/Relative_luminance) in colorimetric spaces, using a simplified version of the luminosity function.

<p align="center">
  <img src="./images/Luminosity.svg">
</p>

> Green light contributes the most to the intensity perceived by humans, and blue light the least.

This function returns an integer in the range from 0 to 255, we assign a character according to density to show more colored surface for areas with more intense color (highest values).

```python
CHARS_LIGHT 	= [' ', ' ', '.', ':', '!', '+', '*', 'e', '$', '@', '8']
CHARS_COLOR 	= ['.', '*', 'e', 's', '@']
CHARS_FILLED    = ['░', '▒', '▓', '█']
```

The reduced range of colors supported by the terminal is a problem we need to account for. Modern terminals support up to 256 colors, so we need to find the closest 8 bit color that matches the original pixel in 16 or 24 bit color, we call this set of 256 colors [ANSI colors](https://stackoverflow.com/questions/4842424/list-of-ansi-color-escape-sequences).

![The Mapping of RGB and ANSI Colors](./images/imgPixelSection.png)

![8 Bits Color Table](./images/8-bit_color_table.png)

Finally, when putting it all together, we will have an appropriate character for each pixel and a new color.

![Frame Image by Characters](./images/imgPixelImage.png)

## Audio Playback Optimization

The audio playback system (`--with-audio` / `-a`) has been reworked to eliminate stuttering and sync issues.

### Problem

The original implementation wrote audio chunks synchronously inside the frame rendering loop. Each call to `stream.write(data)` blocked until PyAudio finished playing that chunk, creating audible gaps whenever frame rendering took longer than expected. Additionally, timing was based on `time.process_time()` (CPU time only), which doesn't reflect real elapsed time and caused video to drift behind audio.

### Solution

**Non-blocking audio via PyAudio callback mode**

Audio playback now runs on a separate thread managed by PyAudio's callback system. The audio subsystem requests data from the WAV file as needed, streaming continuously regardless of how busy the main thread is with frame rendering.

```python
# PyAudio calls this from its own thread whenever it needs more audio data
def audio_callback(in_data, frame_count, time_info, status):
    data = wave_file.readframes(frame_count)
    if len(data) == 0:
        audio_finished.set()
        return (data, pyaudio.paComplete)
    return (data, pyaudio.paContinue)
```

**Wall-clock synchronization with frame skipping**

The video loop now tracks real elapsed time using `time.time()` and compares it against each frame's expected presentation time. If rendering falls behind (frame took too long to process), subsequent frames are skipped until the video catches back up to the audio timeline.

```python
expected_time = counter * time_delta
elapsed = time.time() - playback_start

# Drop this frame if we're already past its presentation time
if with_audio and elapsed > expected_time + time_delta:
    counter += 1
    continue
```

**Larger audio buffer**

The buffer size was increased to 4096 frames (from ~1470 at 44.1kHz/30fps), giving the OS audio subsystem more headroom to absorb timing jitter.

### Result

- Smooth, uninterrupted audio playback
- Video stays in sync with audio even on slower terminals
- Graceful degradation: frames are dropped rather than accumulating lag
- No impact on non-audio playback or file export paths

## Contributors

### Code Contributors

This project exists thanks to all the people who contribute. [[Contribute](./CONTRIBUTING.md)].

<a href="https://github.com/joelibaceta/video-to-ascii/graphs/contributors"><img src="https://opencollective.com/video-to-ascii/contributors.svg?width=890&button=false" /></a>

### Financial Contributors

Become a financial contributor and help us sustain our community. [[Contribute](https://opencollective.com/video-to-ascii/contribute/)].

Or maybe just [buy me a coffee](https://ko-fi.com/joelibaceta).

#### Individuals

<a href="https://opencollective.com/video-to-ascii#backers" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/individuals.svg?width=890"></a>

#### Organizations

Support this project with your organization. Your logo will show up here with a link to your website. [[Contribute](https://opencollective.com/video-to-ascii/contribute)]

<a href="https://opencollective.com/video-to-ascii/organization/0/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/0/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/1/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/1/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/2/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/2/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/3/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/3/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/4/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/4/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/5/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/5/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/6/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/6/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/7/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/7/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/8/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/8/avatar.svg"></a>
<a href="https://opencollective.com/video-to-ascii/organization/9/website" target="_blank" rel="noopener"><img src="https://opencollective.com/video-to-ascii/organization/9/avatar.svg"></a>

## As Seen On
<a href="https://www.producthunt.com/posts/video-to-ascii" target="_blank" rel="noopener"><img src="https://user-images.githubusercontent.com/864790/124545434-a2e7fe80-ddee-11eb-9d80-f24049524fd9.png" width="100px"></a>
