#!/usr/bin/env python3
"""
Daily podcast generation script.

Modes (selected with --mode):
  audio  - read input/narration.txt, call OpenAI TTS in chunks (to stay
           under the API's per-request input limit), concatenate the audio,
           enforce the duration limit (speeding up slightly if needed), and
           write output/<file>.mp3 and output/episode-meta.json.
  prune  - delete old GitHub Release assets beyond EPISODES_TO_KEEP.
  feed   - rebuild docs/feed.xml and docs/podcast-history.json from the
           current GitHub Release assets.
"""

import argparse
import datetime
import email.utils
import json
import os
import re
import subprocess
import sys
import xml.sax.saxutils as saxutils

NARRATION_PATH = "input/narration.txt"
OUTPUT_DIR = "output"
META_PATH = os.path.join(OUTPUT_DIR, "episode-meta.json")
DOCS_DIR = "docs"
FEED_PATH = os.path.join(DOCS_DIR, "feed.xml")
HISTORY_PATH = os.path.join(DOCS_DIR, "podcast-history.json")

TTS_MODEL = "gpt-4o-mini-tts"
TTS_URL = "https://api.openai.com/v1/audio/speech"

# OpenAI's TTS endpoint rejects requests whose input exceeds ~2000 tokens.
# Czech text tokenizes at roughly 1.5-2 tokens per word, so we keep each
# chunk well under that ceiling by limiting words per chunk.
CHUNK_MAX_WORDS = 700


def env(name, default=None, required=False):
    value = os.environ.get(name, default)
    if required and not value:
        sys.exit(f"Missing required environment variable: {name}")
    return value


def today_str():
    return datetime.date.today().isoformat()


def word_count(text):
    return len(text.split())


def split_into_chunks(text, max_words=CHUNK_MAX_WORDS):
    """Split narration into TTS-sized chunks without cutting mid-sentence."""
    sentences = re.split(r"(?<=[.!?])\s+", text.strip())
    chunks = []
    current = []
    current_words = 0
    for sentence in sentences:
        if not sentence:
            continue
        w = word_count(sentence)
        if current and current_words + w > max_words:
            chunks.append(" ".join(current))
            current = [sentence]
            current_words = w
        else:
            current.append(sentence)
            current_words += w
    if current:
        chunks.append(" ".join(current))
    return [c for c in chunks if c.strip()]


def ffprobe_duration(path):
    result = subprocess.run(
        [
            "ffprobe", "-v", "error",
            "-show_entries", "format=duration",
            "-of", "default=noprint_wrappers=1:nokey=1",
            path,
        ],
        capture_output=True, text=True, check=True,
    )
    return float(result.stdout.strip())


def speed_up(src_path, dst_path, factor):
    subprocess.run(
        [
            "ffmpeg", "-y", "-i", src_path,
            "-filter:a", f"atempo={factor:.4f}",
            "-vn", dst_path,
        ],
        check=True, capture_output=True,
    )


def call_openai_tts(narration_text, voice, api_key, out_path):
    import requests

    response = requests.post(
        TTS_URL,
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        json={
            "model": TTS_MODEL,
            "voice": voice,
            "input": narration_text,
            "response_format": "mp3",
        },
        timeout=180,
    )
    if response.status_code != 200:
        sys.exit(
            f"OpenAI TTS request failed ({response.status_code}): {response.text[:500]}"
        )
    with open(out_path, "wb") as f:
        f.write(response.content)


def synthesize_full_narration(narration, voice, api_key, raw_path):
    chunks = split_into_chunks(narration)
    print(f"Narration split into {len(chunks)} TTS chunk(s)")

    if len(chunks) == 1:
        call_openai_tts(chunks[0], voice, api_key, raw_path)
        return

    chunk_paths = []
    for i, chunk in enumerate(chunks):
        chunk_path = os.path.join(OUTPUT_DIR, f"chunk_{i:02d}.mp3")
        print(f"Synthesizing chunk {i + 1}/{len(chunks)} ({word_count(chunk)} words)")
        call_openai_tts(chunk, voice, api_key, chunk_path)
        chunk_paths.append(chunk_path)

    concat_list_path = os.path.join(OUTPUT_DIR, "concat_list.txt")
    with open(concat_list_path, "w", encoding="utf-8") as f:
        for p in chunk_paths:
            f.write(f"file '{os.path.abspath(p)}'\n")

    subprocess.run(
        [
            "ffmpeg", "-y", "-f", "concat", "-safe", "0",
            "-i", concat_list_path,
            "-c:a", "libmp3lame", "-q:a", "2",
            raw_path,
        ],
        check=True, capture_output=True,
    )

    for p in chunk_paths:
        os.remove(p)
    os.remove(concat_list_path)


def mode_audio():
    if not os.path.exists(NARRATION_PATH):
        sys.exit(f"Narration file not found: {NARRATION_PATH}")

    with open(NARRATION_PATH, "r", encoding="utf-8") as f:
        narration = f.read().strip()

    if not narration:
        sys.exit("Narration file is empty. Nothing to publish today.")

    max_words = int(env("MAX_NARRATION_WORDS", "1200"))
    wc = word_count(narration)
    if wc > max_words:
        sys.exit(
            f"Narration has {wc} words, which exceeds MAX_NARRATION_WORDS "
            f"({max_words}). Shorten the narration instead of raising this limit."
        )

    api_key = env("OPENAI_API_KEY", required=True)
    voice = env("OPENAI_TTS_VOICE", "cedar")

    os.makedirs(OUTPUT_DIR, exist_ok=True)
    raw_path = os.path.join(OUTPUT_DIR, "raw.mp3")
    synthesize_full_narration(narration, voice, api_key, raw_path)

    duration = ffprobe_duration(raw_path)
    max_duration = float(env("MAX_DURATION_SECONDS", "600"))
    max_speedup = float(env("MAX_SPEEDUP", "1.15"))

    date = today_str()
    prefix = env("EPISODE_PREFIX", "daily-investing")
    filename = f"{prefix}-{date}.mp3"
    final_path = os.path.join(OUTPUT_DIR, filename)

    if duration <= max_duration:
        os.replace(raw_path, final_path)
        final_duration = duration
    else:
        needed_speedup = duration / max_duration
        if needed_speedup > max_speedup:
            sys.exit(
                f"Episode is {duration:.0f}s, which is {needed_speedup:.2f}x over the "
                f"{max_duration:.0f}s limit. That exceeds MAX_SPEEDUP ({max_speedup}). "
                "Shorten the narration script rather than speeding up further."
            )
        speed_up(raw_path, final_path, needed_speedup)
        final_duration = ffprobe_duration(final_path)
        os.remove(raw_path)

    title = f"{env('PODCAST_TITLE', 'Podcast')} – {date}"
    meta = {
        "filename": filename,
        "date": date,
        "title": title,
        "duration_seconds": round(final_duration),
        "word_count": wc,
        "pub_date_rfc822": email.utils.format_datetime(
            datetime.datetime.now().astimezone()
        ),
    }
    with open(META_PATH, "w", encoding="utf-8") as f:
        json.dump(meta, f, indent=2, ensure_ascii=False)

    print(f"Generated {filename} ({final_duration:.0f}s, {wc} words)")


def gh_json(args):
    result = subprocess.run(
        ["gh"] + args + ["--json", "assets"],
        capture_output=True, text=True,
    )
    if result.returncode != 0:
        return {"assets": []}
    return json.loads(result.stdout)


def mode_prune():
    tag = env("RELEASE_TAG", "daily-podcast")
    repo = env("GITHUB_REPOSITORY", required=True)
    keep = int(env("EPISODES_TO_KEEP", "7"))

    data = gh_json(["release", "view", tag, "--repo", repo])
    assets = sorted(data.get("assets", []), key=lambda a: a["name"], reverse=True)

    for asset in assets[keep:]:
        print(f"Deleting old release asset: {asset['name']}")
        subprocess.run(
            ["gh", "release", "delete-asset", tag, asset["name"],
             "--repo", repo, "--yes"],
            check=False,
        )


def rss_escape(text):
    return saxutils.escape(text or "")


def mode_feed():
    tag = env("RELEASE_TAG", "daily-podcast")
    repo = env("GITHUB_REPOSITORY", required=True)
    keep = int(env("EPISODES_TO_KEEP", "7"))
    title = env("PODCAST_TITLE", "Podcast")
    author = env("PODCAST_AUTHOR", "")
    description = env("PODCAST_DESCRIPTION", "")

    data = gh_json(["release", "view", tag, "--repo", repo])
    assets = sorted(data.get("assets", []), key=lambda a: a["name"], reverse=True)[:keep]

    owner, repo_name = repo.split("/")
    pages_url = f"https://{owner}.github.io/{repo_name}/"

    today_meta = {}
    if os.path.exists(META_PATH):
        with open(META_PATH, "r", encoding="utf-8") as f:
            today_meta = json.load(f)

    history = []
    if os.path.exists(HISTORY_PATH):
        with open(HISTORY_PATH, "r", encoding="utf-8") as f:
            try:
                history = json.load(f)
            except json.JSONDecodeError:
                history = []

    by_filename = {item["filename"]: item for item in history}
    if today_meta.get("filename"):
        by_filename[today_meta["filename"]] = today_meta

    episodes = []
    for asset in assets:
        name = asset["name"]
        record = by_filename.get(name, {
            "filename": name,
            "date": name,
            "title": f"{title} – {name}",
            "duration_seconds": None,
            "word_count": None,
            "pub_date_rfc822": email.utils.format_datetime(
                datetime.datetime.now().astimezone()
            ),
        })
        record["url"] = f"https://github.com/{repo}/releases/download/{tag}/{name}"
        record["size_bytes"] = asset.get("size", 0)
        episodes.append(record)

    with open(HISTORY_PATH, "w", encoding="utf-8") as f:
        json.dump(episodes, f, indent=2, ensure_ascii=False)

    items_xml = []
    for ep in episodes:
        items_xml.append(f"""    <item>
      <title>{rss_escape(ep.get('title'))}</title>
      <description>{rss_escape(description)}</description>
      <enclosure url="{rss_escape(ep.get('url'))}" length="{ep.get('size_bytes', 0)}" type="audio/mpeg"/>
      <guid isPermaLink="false">{rss_escape(ep.get('filename'))}</guid>
      <pubDate>{rss_escape(ep.get('pub_date_rfc822'))}</pubDate>
      <itunes:duration>{ep.get('duration_seconds') or ''}</itunes:duration>
      <itunes:explicit>false</itunes:explicit>
    </item>""")

    feed = f"""<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{rss_escape(title)}</title>
    <link>{rss_escape(pages_url)}</link>
    <atom:link href="{rss_escape(pages_url)}feed.xml" rel="self" type="application/rss+xml"/>
    <description>{rss_escape(description)}</description>
    <language>cs</language>
    <itunes:author>{rss_escape(author)}</itunes:author>
    <itunes:explicit>false</itunes:explicit>
    <itunes:category text="Business"/>
{chr(10).join(items_xml)}
  </channel>
</rss>
"""

    os.makedirs(DOCS_DIR, exist_ok=True)
    with open(FEED_PATH, "w", encoding="utf-8") as f:
        f.write(feed)

    print(f"Wrote {FEED_PATH} with {len(episodes)} episodes")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--mode", required=True, choices=["audio", "prune", "feed"])
    args = parser.parse_args()

    if args.mode == "audio":
        mode_audio()
    elif args.mode == "prune":
        mode_prune()
    elif args.mode == "feed":
        mode_feed()


if __name__ == "__main__":
    main()
