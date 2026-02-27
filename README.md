<p align="center">
  <img src="https://i.imgur.com/Q94CDKC.png" width="200px" alt="iYTDL Logo">
</p>

<h1 align="center">iYTDL</h1>

<p align="center">
  <b>Async Inline YouTube-DL for Pyrogram-based Telegram Bots</b>
</p>

<p align="center">
  <a href="https://pypi.org/project/iytdl/"><img alt="PyPI Version" src="https://img.shields.io/pypi/v/iytdl?color=blue&style=flat-square"></a>
  <a href="https://pepy.tech/project/iytdl"><img alt="Downloads" src="https://static.pepy.tech/badge/iytdl?style=flat-square"></a>
  <a href="https://github.com/iytdl/iytdl/actions"><img alt="CI" src="https://img.shields.io/github/actions/workflow/status/iytdl/iytdl/pypi-publish.yaml?style=flat-square&label=CI"></a>
  <a href="https://github.com/iytdl/iytdl/blob/master/LICENSE"><img alt="License: GPLv3" src="https://img.shields.io/badge/License-GPLv3-blue.svg?style=flat-square"></a>
  <a href="https://github.com/psf/black"><img alt="Code Style: Black" src="https://img.shields.io/badge/code%20style-black-000000.svg?style=flat-square"></a>
  <a href="https://www.python.org/"><img alt="Python 3.8+" src="https://img.shields.io/badge/python-3.8%2B-blue?style=flat-square&logo=python&logoColor=white"></a>
</p>

<p align="center">
  A production-ready Python library that wraps <a href="https://github.com/yt-dlp/yt-dlp">yt-dlp</a> with an async, inline-friendly interface for <a href="https://docs.pyrogram.org/">Pyrogram</a> Telegram bots — search, download, and upload media from <a href="https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md">1000+ sites</a> directly inside Telegram.
</p>

---

## Highlights

| | Feature | Details |
|---|---|---|
| ⚡ | **Fully Async** | Built on `asyncio`, `aiohttp`, and `aiosqlite` — non-blocking from search to upload |
| 🗃️ | **Smart Caching** | SHA-1 hashed SQLite cache eliminates duplicate downloads and speeds up repeat queries |
| 🌐 | **1000+ Sites** | Powered by yt-dlp — YouTube, Dailymotion, Vimeo, SoundCloud, Twitter, and [many more](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md) |
| 🎛️ | **Quality Selector** | Inline buttons let users pick video resolution (144p → 1440p) or audio bitrate (128–320 kbps) |
| 📤 | **Telegram Upload** | Uploads audio/video to Telegram with real-time progress bars, metadata, and thumbnails |
| 🔌 | **External Downloader** | Optional [Aria2c](https://aria2.github.io/) integration for faster, resumable downloads |
| 🖼️ | **Thumbnails** | Auto-fetches best thumbnail, uploads to Telegraph, and embeds album art for audio |
| 🧹 | **Context Manager** | `async with iYTDL(...) as ytdl:` — clean resource management out of the box |

---

## Architecture

```
src/iytdl/
├── main.py             # iYTDL — primary class (inherits Extractor + Downloader + Uploader)
├── extractors.py       # yt-dlp format extraction & inline button generation
├── downloader.py       # async video/audio downloading with progress hooks
├── upload_lib/
│   ├── uploader.py     # Telegram upload with progress tracking
│   ├── progress.py     # upload progress bar renderer
│   └── functions.py    # media metadata helpers (ffprobe, thumbnails)
├── sql_cache.py        # aiosqlite-backed search/URL cache
├── formatter.py        # search result → Telegram message formatter
├── processes.py        # cancellable download/upload process manager
├── types/              # SearchResult, ExternalDownloader, Aria2c dataclasses
├── constants.py        # URLs, regex patterns
├── exceptions.py       # NoResultFoundError, DownloadFailedError
└── utils.py            # run_sync, humanbytes, time_formater, telegraph upload
```

---

## Quick Start

### Installation

```bash
pip install iytdl
```

### Minimal Example

```python
from iytdl import iYTDL

async def main():
    async with iYTDL(log_group_id=LOG_GROUP_ID, cache_path="cache") as ytdl:

        # 🔍 Search YouTube
        result = await ytdl.search("never gonna give you up")
        print(result.msg)

        # 🔗 Parse any supported URL
        result = await ytdl.parse("https://www.youtube.com/watch?v=dQw4w9WgXcQ")

        # ⬇️ Download video
        key = await ytdl.download(
            url="https://www.youtube.com/watch?v=dQw4w9WgXcQ",
            uid="mp4",
            downtype="video",
            update=message,  # Pyrogram Message or CallbackQuery
        )

        # 📤 Upload to Telegram
        await ytdl.upload(client, key, "video", message)
```

---

## API Reference

### `iYTDL` — Main Class

```python
iYTDL(
    log_group_id: Union[int, str],          # Telegram chat ID for uploading media
    session: Optional[ClientSession],        # aiohttp session (auto-created if None)
    silent: bool = False,                    # suppress yt-dlp stdout
    download_path: str = "downloads",        # local download directory
    cache_path: str = "",                    # SQLite cache directory
    delete_media: bool = False,              # auto-cleanup after upload
    external_downloader: Optional[ExternalDownloader] = None,  # e.g. Aria2c(...)
    ffmpeg_location: str = "ffmpeg",         # custom ffmpeg path
)
```

### Core Methods

| Method | Description |
|---|---|
| `search(query)` | Search YouTube, returns paginated `SearchResult` with inline buttons |
| `parse(search_query, extract=True)` | Auto-detect: YouTube URL → extract, generic URL → extract, text → search |
| `next_result(key, index)` | Navigate cached search results (pagination) |
| `extract_info_from_key(key)` | Get download buttons for a cached URL/video ID |
| `download(url, uid, downtype, update)` | Download media with live progress bar in Telegram |
| `upload(client, key, downtype, update)` | Upload downloaded media to Telegram with progress |
| `get_download_button(yt_id)` | Generate quality-selection inline keyboard for a YouTube video |
| `listview(key)` | Grid view of all cached search results |
| `paste_to_tg(title, content)` | Paste content to Telegraph |
| `get_ytthumb(yt_id)` | Fetch best available YouTube thumbnail |

### Callback Patterns (for `CallbackQueryHandler`)

```python
# Pagination (Back / Next)
r"^yt_(back|next)\|(?P<key>[\w-]{5,11})\|(?P<pg>\d+)$"

# List all results
r"^yt_listall\|(?P<key>[\w-]{5,11})$"

# Extract info
r"^yt_extract_info\|(?P<key>[\w-]{5,11})$"

# Download (generic + YouTube)
r"yt_(?P<mode>gen|dl)\|(?P<key>[\w-]+)\|(?P<choice>[\w-]+)\|(?P<dl_type>a|v)$"

# Cancel download/upload
r"^yt_cancel\|(?P<process_id>[\w\.]+)$"
```

---

## External Downloader (Aria2c)

```python
from iytdl.types.external_downloader import Aria2c

async with iYTDL(
    log_group_id=LOG_GROUP_ID,
    external_downloader=Aria2c(executable_path="/usr/bin/aria2c"),
    cache_path="cache",
) as ytdl:
    ...
```

---

## Requirements

| Dependency | Purpose |
|---|---|
| [Python](https://www.python.org/) ≥ 3.8 | Runtime |
| [yt-dlp](https://github.com/yt-dlp/yt-dlp) | Media extraction & download engine |
| [Pyrogram](https://docs.pyrogram.org/) | Telegram Bot framework |
| [FFmpeg](http://ffmpeg.org/) | Audio/video post-processing |
| [Aria2c](https://aria2.github.io/) | *(optional)* External download accelerator |

---

## Development

```bash
git clone https://github.com/iytdl/iytdl.git
cd iytdl
poetry install
```

**Run tests:**

```bash
pytest tests/
```

**Pre-commit hooks:**

```bash
pre-commit install
```

**Build wheel:**

```bash
chmod +x scripts/install.sh && ./scripts/install.sh
```

---

## Screenshots

> **Live Demo Bot:** [@iytdl_bot](https://t.me/iytdl_bot)

<p align="center">
  <img src="https://user-images.githubusercontent.com/88159798/136582521-ba5d0c75-5e44-4d2c-8bc7-365a1137d6a9.png" width="30%" />
  <img src="https://user-images.githubusercontent.com/88159798/136582483-2822123c-bb5c-47f3-8a71-5dc0fa0429ba.png" width="30%" />
  <img src="https://user-images.githubusercontent.com/88159798/136582503-c954c731-e0cc-444a-bd8d-9220a4e5e35c.png" width="30%" />
</p>
<p align="center">
  <img src="https://user-images.githubusercontent.com/88159798/136582494-4193b4f2-9db0-4a5f-b799-deb81b7b3245.png" width="30%" />
  <img src="https://user-images.githubusercontent.com/88159798/136582514-bbac6cb4-0a49-4689-9da2-b30abb4de443.png" width="30%" />
  <img src="https://user-images.githubusercontent.com/88159798/136582506-f202cb07-3ce3-480b-8709-8d39f1f04540.png" width="30%" />
</p>
<p align="center">
  <img src="https://user-images.githubusercontent.com/88159798/136582476-dac517f3-34b0-4497-96de-98031ace4a65.png" width="30%" />
</p>

---

## Real-World Usage

**[YouTube.py](https://github.com/code-rgb/droid/blob/master/droid/modules/youtube.py)** — Full bot module built with iYTDL

---

## Tech Stack

<p align="center">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/yt--dlp-FF0000?style=for-the-badge&logo=youtube&logoColor=white" />
  <img src="https://img.shields.io/badge/Pyrogram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white" />
  <img src="https://img.shields.io/badge/SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white" />
  <img src="https://img.shields.io/badge/FFmpeg-007808?style=for-the-badge&logo=ffmpeg&logoColor=white" />
  <img src="https://img.shields.io/badge/Poetry-60A5FA?style=for-the-badge&logo=poetry&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white" />
</p>

---

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

---

<p align="center">
  <sub>Built with ❤️ by <a href="https://github.com/sh-himanshu">Himanshu</a></sub>
</p>
