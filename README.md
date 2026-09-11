# term-spotify

A terminal Spotify player for macOS with colored braille album art, animated EQ visualizer, and full system audio integration.

![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue) ![macOS](https://img.shields.io/badge/platform-macOS-lightgrey) ![Spotify Premium](https://img.shields.io/badge/Spotify-Premium-1DB954)

## Features

- **Braille album art** — full-color album art rendered in Unicode braille, transparent pixels show your terminal background
- **Animated EQ visualizer** — real-time FFT via BlackHole loopback, falls back to Spotify audio analysis API
- **System audio integration** — auto-launches Background Music, routes audio through Multi-Output Device (speakers + BlackHole)
- **CoreAudio volume control** — reads and writes hardware volume directly, bypasses Background Music's software lock
- **Playlist browser** — press `p` to browse all your playlists, type to filter, arrow keys to scroll, enter to play
- **Liked songs shuffle** — starts at a random position in your library, not always track #1
- **Track search** — type a query, plays the top Spotify result
- **Session resume** — `r` picks up the exact track and position from last time
- **Rate-limit aware** — respects Spotify API 429 `Retry-After` headers, shared across all threads

---

## Requirements

### System
- macOS (Apple Silicon or Intel)
- Python 3.9+
- Spotify Premium account
- [spotifyd](https://github.com/Spotifyd/spotifyd) — Spotify Connect daemon
- [Background Music](https://github.com/kyleneideck/BackgroundMusic) — per-app volume and auto-pause
- [BlackHole 2ch](https://github.com/ExistentialAudio/BlackHole) — virtual audio loopback for FFT
- [SwitchAudioSource](https://github.com/deweller/switchaudio-osx) — CLI audio device switching

### Python packages
```bash
pip install spotipy Pillow
pip install sounddevice numpy   # optional — enables real-time FFT
```

---

## Installation

### 1. Install system tools

```bash
brew install spotifyd switchaudio-osx
```

Download and install:
- **Background Music** — https://github.com/kyleneideck/BackgroundMusic/releases
- **BlackHole 2ch** — https://github.com/ExistentialAudio/BlackHole/releases

### 2. Install Python dependencies

```bash
pip install spotipy Pillow
pip install sounddevice numpy   # optional
```

### 3. Configure spotifyd

Create `~/.config/spotifyd/spotifyd.conf`:

```toml
[global]
device_name = "Terminal Player"
backend     = "portaudio"
bitrate     = 320
```

### 4. Create a Multi-Output Device

This routes audio to both your speakers and BlackHole simultaneously.

1. Open **Audio MIDI Setup** (Spotlight → "Audio MIDI Setup")
2. Click **+** in the bottom-left → **Create Multi-Output Device**
3. Check both **MacBook Air Speakers** (or your output) and **BlackHole 2ch**
4. Name it exactly: `Multi-Output Device`

### 5. Create a Spotify Developer App

1. Go to https://developer.spotify.com/dashboard
2. **Create app** — any name/description
3. Add Redirect URI: `http://127.0.0.1:9090/callback`
4. Under **Settings** → copy **Client ID** and **Client Secret**

### 6. Run

```bash
python3 term-spotify
```

On first run you'll be prompted for your Client ID and Secret. A browser tab opens for Spotify authorization — click Agree. Credentials are cached in `~/.config/spotify-chart/` for all future runs.

---

## Controls

| Key | Action |
|-----|--------|
| `space` | Play / pause |
| `l` | Shuffle liked songs (random start position) |
| `r` | Resume last session (exact track + position) |
| `p` | Open playlist browser |
| `/` | Search — type a query, plays top result |
| `→` | Next track |
| `←` | Previous track |
| `↑` / `↓` | System volume ±5% |
| `+` / `-` | System volume ±5% (alternate) |
| `q` | Quit |

### Playlist browser

| Key | Action |
|-----|--------|
| type | Filter playlists by name |
| `↑` / `↓` | Scroll through results |
| `⌫` | Delete filter character |
| `↵` | Play selected playlist |
| `esc` | Cancel |

---

## How it works

### Audio chain

```
Background Music (virtual) → Multi-Output Device → MacBook Air Speakers
                                                 → BlackHole 2ch → FFT
```

term-spotify launches Background Music on start and routes its internal output through Multi-Output Device. This means Background Music's auto-pause feature stays active while BlackHole simultaneously receives the audio stream for FFT visualization. On quit, the original audio routing is restored and Background Music exits.

### Spotify Connect

A local `spotifyd` instance registers as a Spotify Connect device called **Terminal Player**. It appears alongside your phone, laptop, and other devices in Spotify's device picker.

### Volume

Volume is controlled by writing `kAudioHardwareServiceDeviceProperty_VirtualMainVolume` directly to the MacBook Air Speakers CoreAudio device. This bypasses Background Music's software volume lock (which pins system volume to 100%) and matches what the hardware media keys do.

### EQ visualization

If BlackHole is found at startup, FFT bands are computed from the live audio stream (`sounddevice` + `numpy`). Otherwise, the player fetches Spotify's pre-computed audio analysis for each track (`/v1/audio-analysis/{id}`) and derives EQ targets from `segments[].loudness_*` and `pitches[]`, synchronized to `progress_ms`.

---

## File locations

| Path | Purpose |
|------|---------|
| `~/.config/spotify-chart/credentials.json` | Spotify app credentials |
| `~/.config/spotify-chart/.cache` | OAuth token cache |
| `~/.config/spotify-chart/last_session.json` | Last playback state (for `r`) |
| `~/.config/spotifyd/spotifyd.conf` | spotifyd device config |
| `/tmp/spotify-debug.log` | Debug log (overwritten each run) |

---

## License

MIT
