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
| Code letters: Hangul | 13 bits per character; with the address, 381 characters hold about 619 bytes. The sharpest option. |
| Code letters: ASCII | Letters, digits, `-` and `_` only, 6 bits per character. For places that reject Hangul. |
| Address prefix | Defaults to `https://g.yeon.at/?` (19 characters); the `https://` is what turns it into a clickable link in comments, which is subtracted from the photo's budget. Leave it empty to output only the code. |
| Character limit | Defaults to 400 (a DCInside comment). |

## Hosting at g.yeon.at (GitHub Pages + custom domain)

1. Create a public GitHub repository and upload `index.html` and `CNAME` from this folder. `CNAME` contains the single line `g.yeon.at`.
2. In **Settings → Pages**, set Source to `Deploy from a branch` and Branch to `main` / `/(root)`.
3. In the DNS settings for `yeon.at`, add a `CNAME` record with name `g` and value `velvitty.github.io`.
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

---

## Detailed documentation

### Contents

1. [Overview](#overview)
2. [Screens](#screens)
3. [Code and link formats](#code-and-link-formats)
4. [Quality and character count](#quality-and-character-count)
5. [Privacy and requirements](#privacy-and-requirements)
6. [Hosting your own copy](#hosting-your-own-copy)
7. [Under the hood](#under-the-hood)
8. [Limitations](#limitations)
9. [Reading PSNR](#reading-psnr)
10. [Troubleshooting](#troubleshooting)
11. [FAQ](#faq)
12. [Development materials](#development-materials)

### Overview

DCInside comments allow at most 400 characters, cannot carry uploaded images, and do not show `data:` links as pictures. A regular JPG as a `data:` link needs about 800 characters for the file header alone, so it cannot fit in 400.

This page uses a dedicated compression method that shrinks a photo to about 300–620 bytes. It writes the result as Hangul syllables or ASCII letters and digits so it fits in one comment line. The recipient opens the link or pastes the code to see the photo.

- Address: https://g.yeon.at/
- This repository holds only the deployed files (`index.html`, `CNAME`, READMEs). Sources and test material are in [Velvitty/img-dev](https://github.com/Velvitty/img-dev).

### Screens

The page has three tabs.

#### Make (만들기)

1. Drop a photo, click the box to choose one, or paste with Ctrl/⌘+V. Several photos can be added at once.
2. Each photo gets a card: the original on the left, what the recipient will see on the right.
3. The numbers under the card mean:

   | Label | Meaning |
   |---|---|
   | 원본 (Original) | Original file size |
   | 코드 (Code) | Characters holding the photo (without the address) |
   | 전체 (Total) | Characters including the address, against the limit. Green when within the limit. |
   | 압축 단계 (Step) | 0–127. Lower means finer compression. |

4. There are three buttons.
   - **복사 (Copy):** copies the address and code as one line. Where copying is blocked, the text is selected so you can press Ctrl/⌘+C.
   - **보기 탭에서 확인 (Check in View tab):** previews the recipient's screen.
   - **삭제 (Delete):** removes the card.

Changing a setting (code letters, character limit, address prefix) re-compresses every photo. On first open there is one sunset example marked `예시 (example)`.

#### View (보기)

Paste a received code or link and the photo appears.

- Surrounding text is ignored. For Hangul codes the longest run of at least 4 characters is used; for ASCII codes, the longest run after the last `?` or `#`.
- Under the photo you see the code length, the code type and the compression step.
- If the code was cut off or is not a code, the page says so.

Arriving through a link (`https://g.yeon.at/?code`) opens this tab directly.

#### JPG data link

A tool for places with generous limits. It re-encodes the photo as JPG and produces a `data:image/jpeg;base64,…` link.

| Setting | Description |
|---|---|
| Target link length | Finds the highest quality that fits, and only shrinks the image when quality would drop below 35. Buttons for YouTube/Reddit (9,900), Instagram (2,150) and GitHub (65,000), each leaving a little room for wrapping tags. |
| Quality | Encodes once at the chosen quality (5–95). |
| Maximum long side | Original, 1920, 1280, 800 or 480 pixels. |

The result can be copied as a link, an `<img>` tag or Markdown.

### Code and link formats

| Format | Example | Characters for the code | Capacity |
|---|---|---|---|
| Hangul + address (default) | `https://g.yeon.at/?뗈앏괟…` | 381 | about 619 bytes |
| ASCII + address | `https://g.yeon.at/?TqVTweO…` | 381 | about 285 bytes |
| Hangul code only | `뗈앏괟…` | 400 | 650 bytes |
| ASCII code only | `TqVTweO…` | 400 | 300 bytes |

- **Hangul code:** 13 bits per character, about 2.2 times as much as ASCII. Uses the 8,192 characters from `가` (U+AC00) to `쯿` (U+CBFF).
- **ASCII code:** only `A–Z a–z 0–9 - _`. Use it where Hangul is not allowed or where length is counted in bytes.
- **Why `https://`:** DCInside comments only become clickable links when they start with `https://`.
- **Hangul in the address:** the address bar shows Hangul, but the browser sends each syllable as 9 characters such as `%EB%97%88`. 381 syllables come to about 3.4 KB, well under URL length limits.
- **`#` form:** `https://g.yeon.at/#code` also opens.

### Quality and character count

Eight photos compressed into the 381 characters after `https://g.yeon.at/?`. PSNR measures how close the result is to the original; higher is closer (see [Reading PSNR](#reading-psnr)).

| Photo | ASCII code | Hangul code |
|---|---|---|
| Pug on a bed | 25.8 dB | 28.0 dB |
| Misty mountains and a hiker | 26.5 dB | 27.6 dB |
| Person in a field | 24.7 dB | 26.7 dB |
| Bear | 24.2 dB | 26.3 dB |
| Walruses | 22.6 dB | 23.9 dB |
| Pug in a blanket (forest) | 21.9 dB | 23.1 dB |
| Fjord and cliff | 21.1 dB | 22.4 dB |
| Black puppy on wooden floor | 20.8 dB | 22.0 dB |
| **Average** | **23.4 dB** | **25.0 dB** |

Some photos clearly work better than others.

- **Works well:** photos built from large shapes and smooth color changes, such as fog, sky, and close-ups of faces or animals.
- **Works poorly:** photos covered in fine texture, such as wood grain, grass or rock. The texture uses up the characters and the result stays blurry.
- **Crop first:** keeping only the part you want to show makes it much sharper at the same length.
- **Text:** text cannot be read at 400 characters.
- **Raising the limit:** gives a sharper result, but where 1,000 or more characters are allowed, the JPG data link tab is better.

### Privacy and requirements

- **No upload:** photos never leave the browser. Compression and decoding both run in the page's own script.
- **The code is the photo:** nothing is stored on a server, so anyone with the code can see the same photo. Delete the code and the photo is gone.
- **External connections:** only the fonts (Google Fonts) load from outside. Without a connection, default fonts are used.
- **Remembered values:** code letters, address prefix and the last tab are kept in browser storage (localStorage). Photos and codes are not stored.
- **Browsers:** recent Chrome, Edge, Safari and Firefox, including on phones.

### Hosting your own copy

You can copy this repository and host it at your own address.

1. Fork the repository or upload `index.html` to your own repository.
2. In **Settings → Pages**, set the branch to `main` / `(root)`.
3. For your own domain, change the contents of `CNAME` to that domain and add a DNS CNAME record pointing to `your-id.github.io`. Without a domain, `your-id.github.io/repository/` also works; delete the `CNAME` file in that case.
4. When opened over http(s), the page fills the address prefix with its own address (e.g. `https://your-id.github.io/repository/?`). A longer address leaves fewer characters for the photo.

Codes made on any address decode on any other, as long as both use the same version of `index.html`.

### Under the hood

```
photo → 64-pixel grid → YCbCr → whole-image 2D DCT → quantization → arithmetic coding → Hangul or ASCII text
```

1. **Shrink:** the photo is reduced to a grid with a 64-pixel long side. It is halved several times to avoid jagged edges, and transparent areas are filled with white.
2. **Split colors:** one brightness channel (Y) and two color-difference channels (Cb, Cr). The eye is more sensitive to brightness, so color is handled more coarsely.
3. **Frequencies:** instead of JPEG's 8×8 blocks, one DCT covers the whole image. There are no block edges, and low frequencies (large shapes) are stored first.
4. **Quantization:** higher frequencies and color channels are rounded more coarsely. The compression step (0–127) sets the rounding interval.
5. **Arithmetic coding:** each value is split into questions such as "is it zero?" and "is it greater than 1?", and compressed with probabilities that adapt to what came before (an LZMA-style range coder).
6. **Write as text:** bytes are written 13 bits at a time as Hangul syllables, or 6 bits at a time as ASCII.
7. **Fit the limit:** a binary search finds the finest compression step that fits. Then trailing characters are removed for as long as the code still decodes identically.
8. **Decode:** the recipient's browser adds up the cosine waves directly at screen resolution. Because it does not stretch a 64-pixel image, the photo blurs smoothly at any size with no pixel steps.

The first 62 bits of a code (about 5 Hangul characters) hold the version, aspect ratio, compression step, average color and coefficient counts. The bit-level format and parameters are in [img-dev's detailed documentation](https://github.com/Velvitty/img-dev/blob/main/README.en.md#codec-design).

### Limitations

- **Quality:** a blurry preview. Text and fine texture cannot be made out.
- **Ripples:** wave patterns appear near sharp brightness edges, because frequencies were cut off.
- **Ceiling:** the 64-pixel grid is a ceiling, so more characters stop helping beyond a point.
- **Aspect ratio:** photos longer than about 3.8:1 are squeezed to that ratio.
- **Transparency:** transparent areas become white.
- **Animated GIFs:** only the first frame is stored.
- **Different versions:** if a later version changes the compression parameters, older codes may decode differently.
- **DCInside autolinking:** whether DCInside includes the Hangul after `?` in the link has not been verified yet. If the link breaks, use the ASCII code, or have the recipient paste the code into the View tab.

### Reading PSNR

PSNR (peak signal-to-noise ratio) expresses in dB how close the result is to the original.

`PSNR = 10 × log10(255² ÷ mean squared error)`

| PSNR | Impression |
|---|---|
| ∞ | Identical to the original |
| 45 dB and up | Practically indistinguishable |
| 35–40 dB | Typical JPG |
| Around 30 dB | Smearing visible on close inspection |
| Around 25 dB | Blurry but recognizable (Hangul code average) |
| Around 20 dB | Only shapes and colors remain |

- Identical images have zero error, so PSNR is infinite.
- Every 10× increase in error lowers PSNR by exactly 10 dB.
- The figures here compare photos shrunk to 160×120. Comparing at a larger size gives lower values.

### Troubleshooting

| Symptom | Fix |
|---|---|
| "이 글자로는 사진을 만들 수 없습니다" (cannot make a photo from this text) | The code was cut off or other characters got mixed in. Copy the whole line again. |
| The link opens the Make tab with no photo | The comment's autolink may have been cut short. Copy the code from the comment and paste it into the View tab. Senders can use the ASCII code. |
| The copy button does nothing | The text is already selected; press Ctrl/⌘+C. |
| The photo looks nothing like the original | The code may come from a different version of the page. Open it on the same version it was made with. |
| The total is a little under the limit | This is normal. Compression steps come in jumps, so a few characters can be left over. |
| The page returns 404 | Check the repository's Pages settings, the `CNAME` file in the root and the DNS CNAME (`g` → `velvitty.github.io`). The first setup takes a few minutes to propagate. |

### FAQ

**Can the original photo be recovered from a code?**
No. A code only holds as much information as a blurry preview.

**Does the same photo always give the same code?**
In the same browser, yes. Browsers shrink images slightly differently, so codes can differ a little between browsers. But once made, a code decodes to the same photo in any browser.

**What about places that allow more than 400 characters?**
Raising the character limit gives a sharper result. At 1,000 characters or more, the JPG data link tab is better.

**Why is the Hangul code sharper than the ASCII code?**
The same number of characters holds about 2.2 times as much information. An ASCII character is one of 64 values; a Hangul character is one of 8,192.

**The Hangul code looks like strange characters.**
It includes rarely used syllables such as `뗈` and `쯿`. The characters carry no meaning on purpose, and most fonts include all of them.

### Development materials

Sources, the build script, test photos and scripts, and the experiment log are in [Velvitty/img-dev](https://github.com/Velvitty/img-dev). The `index.html` here is a copy of the `dist/` folder that `python3 build.py` produces in img-dev.
