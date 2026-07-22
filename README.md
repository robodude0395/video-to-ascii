# video-to-ascii

A simple Python package to play videos in your terminal using ASCII characters.

## Quickstart

### Requirements

- Python 3
- FFmpeg (for audio support)
- PortAudio (for audio support)
- Linux or macOS

### Installation

```bash
# Basic install
pip install video-to-ascii

# With audio support
pip install video-to-ascii[audio]
```

Or install from source:

```bash
git clone https://github.com/YOUR_USERNAME/video-to-ascii.git
cd video-to-ascii
pip install .[audio]
```

### Usage

```bash
# Play a video in your terminal
video-to-ascii -f myvideo.mp4

# Play with audio
video-to-ascii -f myvideo.mp4 -a

# Choose a render strategy (ascii-color, just-ascii, filled-ascii)
video-to-ascii -f myvideo.mp4 --strategy filled-ascii

# Export to a bash script
video-to-ascii -f myvideo.mp4 -o output.sh
```

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
