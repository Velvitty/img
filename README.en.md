<p align="center"><a href="README.md">한국어</a> · <a href="README.en.md">English</a> · <a href="README.ja.md">日本語</a> · <a href="README.es.md">Español</a></p>

# 400-Char Image Squeezer

A tool that turns a photo into text short enough for **a single comment (under 400 characters)**. It is meant for places like DCInside comments, where messages are short and images cannot be posted. The recipient pastes the text into this page, or opens the link, to see the photo.

- Everything is in one `index.html` file and runs in the browser. No server, no uploads.
- The result is a blurry preview. 400 characters hold only 300–650 bytes, enough to recognize shapes, colors and mood.

## How to use

1. In the **만들기 (Make)** tab, drop a photo, click to choose one, or paste with Ctrl/⌘+V.
2. Press **복사 (Copy)** to copy a single line under 400 characters, such as `https://g.yeon.at/?뗈앏괟샡…`. Paste it into your comment as is.
3. The recipient pastes that line into the browser's address bar to see the photo right away. Pasting it into the **보기 (View)** tab also works, and surrounding text is ignored.

Both `https://g.yeon.at/?code` and `https://g.yeon.at/index.html?code` work.

## Settings

| Setting | Description |
|---|---|
| Code letters: Hangul | 13 bits per character, about 630 bytes in 400 characters. The sharpest option. |
| Code letters: ASCII | Letters, digits, `-` and `_` only, 6 bits per character. For places that reject Hangul. |
| Address prefix | Defaults to `https://g.yeon.at/?` (19 characters); the `https://` is what turns it into a clickable link in comments, which is subtracted from the photo's budget. Leave it empty to output only the code. |
| Character limit | Defaults to 400 (a DCInside comment). |

## Hosting at g.yeon.at (GitHub Pages + custom domain)

1. Create a public GitHub repository and upload `index.html` and `CNAME` from this folder. `CNAME` contains the single line `g.yeon.at`.
2. In **Settings → Pages**, set Source to `Deploy from a branch` and Branch to `main` / `/(root)`.
3. In the DNS settings for `yeon.at`, add a `CNAME` record with name `img` and value `your-id.github.io`.
4. Once DNS has propagated, **Settings → Pages** shows `g.yeon.at` as the custom domain. Turn on **Enforce HTTPS**.

GitHub Pages ignores everything after `?` and serves `index.html`, so no server setup is needed. The page reads the part after `?` and decodes the photo. Hangul is sent as `%EB%…` escapes, and the page turns it back into Hangul.

## How it works

1. The photo is shrunk to a grid with a 64-pixel long side and converted to YCbCr.
2. A whole-image 2D DCT is taken and the coefficients are ordered from low to high frequency.
3. Higher frequencies and the color (chroma) channels are quantized more coarsely.
4. Levels are compressed with a context-adaptive binary arithmetic coder (an LZMA-style range coder).
5. A binary search finds the finest quantizer that fits the limit, and trailing characters are dropped while decoding still matches.
6. The viewer evaluates the cosine series at screen resolution, so the enlarged image stays smooth with no blocky pixels.

## JPG data link tab

Where limits are large (YouTube and Reddit allow 10,000 characters), a regular JPG as a `data:` link is sharper. A JPG header alone is several hundred characters, so it can never fit in 400.

## Notes

- DCInside comments do not display images or `data:` links, so recipients must open this page themselves.
- Hangul code uses 8,192 precomposed syllables starting at `가` (U+AC00). On sites that limit bytes rather than characters, use the ASCII code.
