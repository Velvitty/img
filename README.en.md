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

## How it works

1. The photo is shrunk to a grid with a 64-pixel long side and converted to YCbCr. With a generous character budget the grid grows to 96–512 pixels.
2. A whole-image 2D DCT is taken and the coefficients are ordered from low to high frequency.
3. Higher frequencies and the color (chroma) channels are quantized more coarsely.
4. Levels are compressed with a context-adaptive binary arithmetic coder (an LZMA-style range coder).
5. A binary search finds the finest quantizer that fits the limit, and trailing characters are dropped while decoding still matches.
6. The viewer evaluates the cosine series at screen resolution, so the enlarged image stays smooth with no blocky pixels.

## JPG data link tab

Where `data:` links are shown directly as images (sites that allow HTML or Markdown images), a regular JPG can be used as a `data:` link. At the same character count, this page's code is sharper (a 1200×1200 picture in 7,000 characters: JPG 25.6 dB, Hangul code 28.0 dB). A JPG header alone is several hundred characters, so it can never fit in 400.

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
6. [Opening on other addresses](#opening-on-other-addresses)
7. [Under the hood](#under-the-hood)
8. [Limitations](#limitations)
9. [Reading PSNR](#reading-psnr)
10. [Troubleshooting](#troubleshooting)
11. [FAQ](#faq)
12. [Development materials](#development-materials)
13. [Design notes](#design-notes)
    - [Design notes: problem statement](#design-notes-problem-statement)
    - [Design notes: addresses and transport](#design-notes-addresses-and-transport)
    - [Design notes: static hosting](#design-notes-static-hosting)
    - [Design notes: preprocessing](#design-notes-preprocessing)
    - [Design notes: transform](#design-notes-transform)
    - [Design notes: quantization](#design-notes-quantization)
    - [Design notes: entropy coding](#design-notes-entropy-coding)
    - [Design notes: bitstream format](#design-notes-bitstream-format)
    - [Design notes: text packing](#design-notes-text-packing)
    - [Design notes: rate control and grid selection](#design-notes-rate-control-and-grid-selection)
    - [Design notes: decoding and upscaling](#design-notes-decoding-and-upscaling)
    - [Design notes: page design](#design-notes-page-design)
    - [Design notes: build](#design-notes-build)
    - [Design notes: evaluation method](#design-notes-evaluation-method)
    - [Design notes: the mathematics of PSNR](#design-notes-the-mathematics-of-psnr)
14. [References](#references)

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
   | 격자 (Grid) | The size the photo was stored at: 64px at 400 characters, larger with more characters (up to 512px). |

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
- **`#` form:** `https://g.yeon.at/#code` also opens. When a code is long enough that the address would pass about 7 KB (Hangul codes of roughly 770 characters or more), Copy uses `#` instead of `?` automatically. Everything after `?` is sent to the server, and GitHub Pages rejects addresses over 8 KB (error 414).

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
- **Raising the limit:** the grid holding the photo grows, so the result gets sharper. Measured at 512×512 on a 1200×1200 picture: 400 characters gave a 64px grid and 21.6 dB, 1,367 gave 96px and 24.0 dB, 4,000 gave 192px and 26.6 dB, and 7,000 gave 384px and 28.0 dB. The grid tops out at 512px, so beyond about 10,000 characters there is little gain.

### Privacy and requirements

- **No upload:** photos never leave the browser. Compression and decoding both run in the page's own script.
- **The code is the photo:** nothing is stored on a server, so anyone with the code can see the same photo. Delete the code and the photo is gone.
- **External connections:** only the fonts (Google Fonts) load from outside. Without a connection, default fonts are used.
- **Remembered values:** code letters, address prefix and the last tab are kept in browser storage (localStorage). Photos and codes are not stored.
- **Browsers:** recent Chrome, Edge, Safari and Firefox, including on phones.

### Opening on other addresses

This page only works on `g.yeon.at`.

- If the repository is copied to another address (such as `your-id.github.io`), making and viewing photos both stop, and the page shows only a notice and a button to g.yeon.at.
- If the address carried a code, the button passes that code on to g.yeon.at, so the photo can still be viewed there.
- Opened as a local file or on `localhost`, the page works for testing.

### Under the hood

```
photo → 64–512-pixel grid → YCbCr → whole-image 2D DCT → quantization → arithmetic coding → Hangul or ASCII text
```

1. **Shrink:** the photo is reduced to a grid with a 64-pixel long side. If the budget is more than 64 pixels can use, grids of 96, 128, 192, 256, 384 and 512 pixels are tried in turn, keeping whichever is closer to the original. It is halved several times to avoid jagged edges, and transparent areas are filled with white.
2. **Split colors:** one brightness channel (Y) and two color-difference channels (Cb, Cr). The eye is more sensitive to brightness, so color is handled more coarsely.
3. **Frequencies:** instead of JPEG's 8×8 blocks, one DCT covers the whole image. There are no block edges, and low frequencies (large shapes) are stored first.
4. **Quantization:** higher frequencies and color channels are rounded more coarsely. The compression step (0–127) sets the rounding interval.
5. **Arithmetic coding:** each value is split into questions such as "is it zero?" and "is it greater than 1?", and compressed with probabilities that adapt to what came before (an LZMA-style range coder).
6. **Write as text:** bytes are written 13 bits at a time as Hangul syllables, or 6 bits at a time as ASCII.
7. **Fit the limit:** a binary search finds the finest compression step that fits. Then trailing characters are removed for as long as the code still decodes identically.
8. **Decode:** the recipient's browser adds up the cosine waves directly at screen resolution. Because it does not stretch a grid-sized image, the photo blurs smoothly at any size with no pixel steps.

The start of a code (62 bits, about 5 Hangul characters, for the 64-pixel grid) holds the version, grid size, aspect ratio, compression step, average color and coefficient counts. The bit-level format and parameters are in [img-dev's detailed documentation](https://github.com/Velvitty/img-dev/blob/main/README.en.md#codec-design).

### Limitations

- **Quality:** a blurry preview. Text and fine texture cannot be made out.
- **Ripples:** wave patterns appear near sharp brightness edges, because frequencies were cut off.
- **Ceiling:** the grid tops out at 512 pixels, so beyond about 10,000 characters more characters barely help.
- **Time:** with limits of 7,000 characters or more, compression takes a few seconds.
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

### FAQ

**Can the original photo be recovered from a code?**
No. A code only holds as much information as a blurry preview.

**Does the same photo always give the same code?**
In the same browser, yes. Browsers shrink images slightly differently, so codes can differ a little between browsers. But once made, a code decodes to the same photo in any browser.

**What about places that allow more than 400 characters?**
Raising the character limit grows the grid and gives a sharper result (up to 512 pixels). Use the JPG data link tab where `data:` links are shown as images.

**Why is the Hangul code sharper than the ASCII code?**
The same number of characters holds about 2.2 times as much information. An ASCII character is one of 64 values; a Hangul character is one of 8,192.

**The Hangul code looks like strange characters.**
It includes rarely used syllables such as `뗈` and `쯿`. The characters carry no meaning on purpose, and most fonts include all of them.

### Development materials

Sources, the build script, test photos and scripts, and the experiment log are in [Velvitty/img-dev](https://github.com/Velvitty/img-dev). The `index.html` here is a copy of the `dist/` folder that `python3 build.py` produces in img-dev.

### Design notes

These notes are the same as those attached to each part of the detailed documentation in [Velvitty/img-dev](https://github.com/Velvitty/img-dev). Code paths (`src/`, `lab/`, `build.py`) refer to img-dev.

#### Design notes: problem statement

A DCInside comment cannot exceed 400 characters, cannot carry an attached image, and does not show `data:` links as pictures. Under these conditions the only way to pass on a photo is to write the photo itself as text. This project therefore designs a lossy format that fits in one comment line together with a browser page that creates and decodes it. The compression follows the classic chain of transform coding (DCT), quantization, context-adaptive binary arithmetic coding and text packing, but every stage is re-tuned for a budget of only a few hundred bytes.

The budget starts with how much one character can carry: `log2(alphabet size)` bits. base64url carries `log2 64 = 6` bits, and the 8,192 Hangul syllables used here carry `log2 8192 = 13` bits. The 381 characters left after the 19-character address, at 13 bits each, give 4,953 bits, about 619 bytes, for the whole photo. By Shannon's source coding theorem [7] the average code length cannot fall below the model's entropy. A recognizable picture fits in this budget only with both a model that describes photos with low entropy (a transform and a context model) and a coder that comes close to that entropy (arithmetic coding).

Other routes were weighed. A link to an image host is short but depends on an upload and on a server's lifetime, which conflicts with the goal of a code that is complete without any server. A JPEG `data:` link spends about 600 bytes on quantization and Huffman tables alone, more than 800 characters in base64 [3][4][26]. BlurHash [33] and ThumbHash [34] are built for placeholders of a few dozen characters with a fixed number of components (BlurHash allows at most 9 per axis), so they cannot use a 400-character budget. Hence a dedicated codec whose quality keeps growing as the budget grows.

The design brings no server cost and no privacy risk, and as long as the text is copied it decodes to the same photo anywhere. In return the information ceiling is hard: at 400 characters the result cannot go beyond a blurry preview. Whether DCInside autolinks a link containing Hangul all the way through has not been verified yet.

#### Design notes: addresses and transport

The default link has the form `https://g.yeon.at/?code`. DCInside only turns text starting with `https://` into a clickable link, and GitHub Pages ignores everything after `?` and serves the same `index.html`. As a result every address can open a different photo with no server configuration. On load the page reads `location.search`, or `location.hash` when the query is empty, to get the code.

In URI syntax [27] the query after `?` is sent to the server while the fragment after `#` stays in the browser. A Hangul syllable is 3 bytes in UTF-8, and each byte becomes the three characters `%XX` in an address, so one syllable takes 9 bytes. A 381-character code therefore takes 381 × 9 = 3,429 bytes plus the 19-byte address. An HTTP server may answer 414 to an overly long target [28], and direct requests to g.yeon.at were accepted at 6,000 bytes and rejected with 414 at 9,000. To leave room for the rest of the request line, Copy replaces `?` with `#` once the URL-encoded address passes 7,000 bytes, which corresponds to `(7000 − 19) / 9 ≈ 775` syllables.

Using `#` all the time would remove any worry about 414, but `?` is the default form and is enough for short links. So 400-character links keep the `?` form, while links of thousands of characters switch to `#` and open without errors; the part after `#` never reaches server logs either. The 414 boundary is a measurement of how GitHub Pages and its CDN behave today and may change, and how far an autolinker on a forum or messenger extends over non-ASCII characters differs by service [32].

#### Design notes: static hosting

Compression and decoding both happen in the browser, so the server only has to deliver a file. Static hosting was therefore chosen: only `index.html` and `CNAME` go to GitHub Pages, under the custom domain `g.yeon.at` [32]. Deployment consists of copying the `dist/` folder that `build.py` produces in img-dev into the img repository.

Development material (img-dev) and the deployed copy (img) live in separate repositories. The deploy repository holds only the files the page needs, while sources, test material and by-products stay in the development repository. A serverless function or an own server could handle the address restriction or usage statistics on the server side, at the price of running costs and the burden of handling data. Static hosting fits the goal most simply: no running cost, and GitHub Pages as the only point of failure. In exchange, checks such as the address restriction can only run inside the browser.

#### Design notes: preprocessing

Preprocessing shrinks the photo to the grid size, composites it on white and converts it to YCbCr. Sizing the transform to the budget keeps computation low, and separating brightness from color lets color be treated more coarsely. Color conversion uses the full-range BT.601 coefficients of JFIF (`Y = 0.299R + 0.587G + 0.114B` and so on) [5][6]. Transparent areas are filled with white.

Downscaling is bound by sampling theory. By the sampling theorem [36], shrinking by a factor k folds frequencies above `1/(2k)` of the original resolution back into the image as aliasing. The browser's bilinear filter only looks at a 2×2 neighborhood, so one large reduction is effectively point sampling and cannot avoid the problem. The photo is therefore halved on a canvas until it is 2–4 times the target and then drawn once more at grid size. Each step comes close to a 2×2 average, giving the same low-pass effect as a mipmap pyramid [18][30].

The aspect ratio is stored in 6 bits as `a = round(16 · log2(width/height))`. With 16 steps per octave the ratio error is at most `2^(1/32) ≈ 2.2%`, and ±31 steps cover up to about 3.83:1. A 64-pixel grid has 64 × 48 = 3,072 coefficients per channel, enough for the few hundred non-zero ones a 400-character budget (about 619 bytes) can hold, while the transform finishes within a millisecond.

Color is kept separate because the eye is less sensitive to color contrast than to brightness contrast [17]. Compressing RGB directly wastes bits on information repeated across the three channels. Instead of reducing color spatially as in 4:2:0, the color channels use a 1.7× larger quantization step on the same grid. With a whole-image DCT the high-frequency color coefficients end up at zero anyway, so the effect is similar and the implementation is simpler.

Photos of any size thus follow the same path, and no staircase artifacts from downscaling leak into the result. Canvas downscaling does differ slightly between browsers, so the same photo can yield slightly different codes in different browsers; photos longer than 3.8:1 are squeezed to that ratio, and transparency is lost to white.

#### Design notes: transform

Neighboring pixels in natural images are strongly correlated. After a DCT the energy gathers in a few low-frequency coefficients, and the rest can be set to zero with little visible loss. The DCT is known to approach the optimal transform (KLT) for a first-order Markov process and to compact energy well [1][2]. This codec applies one 2D DCT-II to the whole grid with the following normalization.

```math
A[u,v] = \frac{\alpha_u \alpha_v}{WH} \sum_{x=0}^{W-1}\sum_{y=0}^{H-1} f(x,y)\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H},\quad \alpha_0=1,\ \alpha_{k>0}=2
```

```math
f(x,y) = \sum_{u,v} A[u,v]\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H}
```

Under this normalization `A[0,0]` is the mean brightness and every other coefficient is the amplitude of its cosine wave, so quantization steps can be set directly in brightness units. The transform is computed separably, horizontally then vertically, at a cost of `O(W·H·(W+H))`: about 340,000 multiplications for a 64 × 48 grid and about 270 million for 512 × 512.

Coefficients are ordered by increasing `r = √((u·s/W)² + (v·s/H)²)` with `s = √(W·H)`. Using a frequency normalized to a square puts the same visible frequency in the same position for wide and tall photos. The order is cached per grid size, and ties on r are broken by the integers v and u so that browsers and Node.js produce identical orders.

There is a reason for not splitting into 8×8 blocks as JPEG does [3][4]. With a small budget only the DC of each block survives and a checkerboard appears. The original motivations for blocks, computation and local adaptation, matter little at 64–512 grids. Wavelets can preserve edges better at the same budget but make the implementation and context model more complex. A whole-image DCT blurs smoothly without blocks even when only low frequencies remain, which is the right kind of degradation for a tiny budget.

The price is that, being global, fine texture in one area competes with the whole picture for the budget. A truncated cosine series overshoots by about 9% near edges (the Gibbs phenomenon) [19] and leaves ripples. No fast FFT-based DCT [24] is used yet, so a 512 grid takes close to a second.

#### Design notes: quantization

Most of the compression is decided in quantization: the more coarsely the less visible components are discarded, the better the result looks at the same length. The step size and rounding rule of this codec are:

```math
\mathrm{step} = 0.0002 \cdot 1.06^{q} \cdot (1 + 0.11\,r) \cdot c_{\mathrm{chroma}} \cdot s_{\mathrm{grid}},\qquad \mathrm{level} = \mathrm{sign}(x)\left\lfloor \frac{|x|}{\mathrm{step}} + 0.5 - 0.1 \right\rfloor
```

Here `c = 1.7` for color channels and `s = 64/long side` in version 1 (grids of 96 pixels or more); reconstruction is `level · step`.

DCT AC coefficients are known to follow roughly a Laplacian distribution [15], and for Laplacian sources a uniform quantizer with a widened zero bin (dead zone) is favorable in rate-distortion terms [14]. In the rule above a coefficient becomes zero below `0.6 · step`. The frequency weight `1 + 0.11r` is a straight-line approximation of the contrast sensitivity function [16], and the 1.7× color factor reflects the lower sensitivity to color contrast [17]. The values were chosen by comparing PSNR on the test photos (item 4 of [Experiments and decisions](https://github.com/Velvitty/img-dev/blob/main/README.en.md#experiments-and-decisions)). q is exponential so that a range of `1.06^127 ≈ 1,620`× is divided evenly in 7 bits. Each step changes the step size by 6%, so the rate can be adjusted smoothly, which keeps fitting the character limit simple.

The grid factor `s = 64/long side` follows from this relation. Texture with per-pixel variance σ² has coefficient variance σ² under an orthonormal DCT. The amplitude coefficients used here are the orthonormal ones multiplied by `2/√(WH)`, so their standard deviation shrinks as `1/√(WH)`, which at a fixed aspect ratio is proportional to `1/long side`. Multiplying the step by `64/long side` makes the same q mean the same relative precision on every grid. Without this factor fine texture was still rounded to zero on large grids, and the 256-pixel grid stopped at 1,749 characters.

JPEG-style 8×8 quantization tables are tied to the block size and do not suit a whole-image DCT or a changing grid. Pulling reconstruction values toward zero is better in theory for Laplacian sources, but leaving them unshifted measured best. The weights were chosen by PSNR on eight test photos and may differ from the optimum for perceived quality. In version 0 the DC uses 6 bits for brightness and 5 for color, so the average color can be off by up to about 2/255 (brightness) and 4/255 (color); version 1 raises this to 8 bits.

#### Design notes: entropy coding

Most quantized coefficients are zero, and the chance of a coefficient being zero varies strongly with its frequency and neighbors. Only when that variation is reflected in the probabilities does the length come close to the entropy. The integers are therefore not written out directly but coded with a binary arithmetic coder that uses context-dependent probabilities.

An event of probability p costs `−log2 p` bits, and arithmetic coding makes the total only a few bytes longer than that sum [8]. Huffman coding spends at least one bit per symbol, even on a "zero" with probability 0.95, where arithmetic coding spends about 0.074 bits. The spare bytes left at the end are removed by the trailing-character trim described in [Fitting the character limit](https://github.com/Velvitty/img-dev/blob/main/README.en.md#fitting-the-character-limit).

Each coefficient is split into binary decisions in the order zero or not, sign, greater than 1, greater than 2, and Exp-Golomb, the same binarization as CABAC in H.264 [11][12][13]. Each decision is coded with the probability of a context defined by the channel, the frequency band and whether the previous two coefficients were zero. The coder follows the range coder of the LZMA SDK, with a 32-bit range and carry handling through a cache byte [9][10].

Probabilities are 11-bit integers updated per bit as `p ← p + (2048 − p)/16` on 0 and `p ← p − p/16` on 1. This exponential moving average has an effective memory of about 16 bits, so it adapts quickly even within a stream of a few hundred bytes; on the validation photos 1/16 beat LZMA's default of 1/32 by 0.04 dB. The model is kept small at 216 contexts, because with little data too many contexts would end the stream before each had learned anything. Starting probabilities trained on the test photos gained only 0.03–0.07 dB, too little to justify shipping a table in the code. The header goes through the same coder at probability 0.5 and costs exactly one bit per bit with no separate bit-packing code.

Written this way, the same information takes far fewer bits than with fixed-length codes, so a 400-character code holds hundreds of coefficients. Weaknesses remain: signs still cost almost a full bit, two-dimensional neighbors such as the coefficient above are not used as context, no stronger model such as context mixing is used, and damage in the middle of a stream corrupts everything after it.

#### Design notes: bitstream format

The decoder must rebuild the same grid, coefficient order and step sizes from the code alone. The code therefore starts with a header holding the version, grid, aspect ratio, compression step, average color and coefficient counts, followed by the coefficients.

The version 0 header is 62 bits, 1.2% of a 400-character budget of about 5,200 bits. Storing the coefficient counts in the header instead of an end-of-block marker costs a fixed 10–16 bits per channel, with no per-coefficient question of whether the block ends here [11]. Version 1 adds a 3-bit grid index after the version bit, uses 8 bits per DC value and `⌈log2(W·H)⌉` bits per coefficient count.

Backward compatibility was the main concern when extending the format. The 64-pixel grid is still written as version 0, so codes already in circulation and 400-character codes did not change by a single bit (checked with 16 codes from the test photos). A multi-bit version field from the start would have been tidier, but at 400 characters every bit counts, so it started as one bit. Both values are now in use, so the next extension will use the unused grid index 7 of version 1 as a marker. The remaining weakness is the lack of a checksum: truncated or altered codes cannot be reliably detected and may decode into a wrong picture.

#### Design notes: text packing

Comments are limited by character count, so each character must carry as many bits as possible. The bytes are joined into a bit string and cut into 13-bit groups written as the Hangul syllable `U+AC00 + value`, or into 6-bit groups written as base64url characters [25]. Characters outside these ranges are ignored when reading.

Unicode has 11,172 Hangul syllables at `U+AC00–U+D7A3` [29]. Using all of them would carry `log2 11172 ≈ 13.447` bits, 3.4% more than 13. That, however, would require treating the whole code as one big number and converting its base, an `O(n²)` computation, whereas with a power of two, 8,192 characters, bits can simply be cut and joined. Spare bits in the last character are zero-filled, which matches the decoder's rule of reading missing bytes as zero and makes the trailing-character trim possible.

Other alphabets were considered. base83 (BlurHash [33]) or base85 carry slightly more than 6 bits but include address-special characters such as `#`, `%` and `?`. Emoji or Han characters could carry more bits, but their display and input are unreliable. Hangul is natural in DCInside comments, present in most fonts, and holds about 2.2 times as much information as ASCII in the same 400 characters.

Where length is counted in UTF-8 bytes, however, Hangul carries 4.33 bits per byte, worse than ASCII's 6. A normalization that decomposes syllables into jamo (NFD) [29] also breaks a code, and the decoder does not currently convert pasted text back to NFC.

#### Design notes: rate control and grid selection

Only a character limit is given, and within it the quality should be as high as possible. To achieve this, the compression step q and the grid size are chosen together.

The code length R(q) decreases almost monotonically with q, so a binary search over 0–127 finds the smallest q with `R(q) ≤ budget` in about `⌈log2 128⌉ = 7` encodes. Trailing characters are then removed one at a time for as long as the decoded result stays the same. This removes the spare bytes the range coder emits at the end, and since it is checked by actually decoding, it cannot go wrong.

Whether a given rate is better spent on resolution or on quantization precision is a classic rate-distortion optimization problem [20]. It is solved here with a greedy rule. The photo is first encoded on the 64-pixel grid. If even q = 0 leaves characters unused, that grid cannot fill the budget, so the photo is re-encoded on the next grid size. Both results are compared by mean squared error against the original at the larger grid size and the better one is kept; the search stops when the larger grid is worse or no longer reaches q = 0 [8]. A grid that fits at q = 0 skips the binary search, which cut a 7,000-character encode from about 1.8 s to 1.4 s.

Encoding every grid and comparing them all might find better results, but large grids take seconds each; a 768-pixel grid was tried and dropped because 20,000 characters took 16 seconds. The outcome is that 400 characters remain as fast as before, while 7,000 characters use 6,968 of them and reach 28.0 dB. R(q) is not strictly monotonic, so a few characters may be left over; the greedy rule does not guarantee the optimal grid; and the computation runs on the main thread, so the page pauses briefly for large budgets.

#### Design notes: decoding and upscaling

Building a grid-size image and then stretching it would add pixel steps or interpolation blur, so the decoder computes the screen-size image directly from the coefficients. Screen coordinates, converted to grid units, are substituted for x and y in the inverse transform and the cosine series is summed as is. The inverse DCT is a smooth function defined for non-integer x as well, so sampling it at any resolution amounts to band-limited interpolation in the cosine basis.

Rows are computed first, and rows whose coefficients are all zero are skipped, for a cost of `O(H·W·outW + rows·outH·outW)`. The same function draws card previews (480px long side) and the View tab (960px), ending with a YCbCr-to-RGB conversion clamped to 0–255. An inverse DCT at grid size followed by canvas scaling would be faster, but on small grids it shows the blurry cross patterns typical of bilinear interpolation.

This approach turns very little information into a pleasant, smoothly blurred image. Gibbs ripples [19] remain visible when enlarged, and because `Math.cos` is implementation-approximated by specification [31], pixel values may differ by about ±1 between browsers.

#### Design notes: page design

The page follows the principle of one HTML file with all JS and CSS inline, running only in the browser. Recipients must be able to open it with no preparation, and deployment should amount to copying one file. The codec source is inserted into the page at build time, and apart from fonts the page works offline. A framework would shorten the interface code but enlarge the file and add external dependencies.

To find the code in pasted text, the longest run of at least four Hangul characters is chosen, or failing that, the longest base64url run after the last `?` or `#` [27]. The code is read even when other words surround it, but a Hangul sentence longer than the code can be picked by mistake.

Every localStorage access is guarded so the page falls back to defaults when storage is blocked, and if the clipboard API is refused, copying falls back to selecting the text. Canvas and image handling follow the HTML standard, under which an animated image draws its default frame [30].

The address restriction makes the page draw only a notice and stop when it is opened outside g.yeon.at, localhost, local files and the claude.ai preview. A reliable restriction belongs on a server, but static hosting has no server code, so this check remains a deterrent that does not stop someone who edits the source.

#### Design notes: build

Node.js tests and the browser must use the same codec file for the tested code to be the shipped code. `build.py` therefore removes only the `module.exports` line from `src/codec.js` and inserts the rest at the page's `/*CODEC*/` placeholder, so the build is reproduced by 30 lines of dependency-free Python. Bundlers such as webpack are heavy for this scale and add runtime code to the output. Build outputs (`dist/`, `build/`) are committed as well so they can be used where no build tools exist, which in turn means every change is built and committed together so source and output never drift apart.

#### Design notes: evaluation method

Choosing parameters, and checking that a change did not break earlier results, requires reproducible numbers. This project compares by PSNR the original box-downscaled to 160×120 with the decoded result drawn at 160×120; large-grid tests compare at 512×512. The original's high frequencies cannot be transmitted anyway, so only a comparison near the size the recipient sees reveals the differences between codecs. PSNR, a logarithm of mean squared error, is simple to compute and easy to reproduce.

The test photos are 8 training and 6 validation images from picsum.photos (Unsplash photos) [35]. After every codec change, the 400-character results, both average PSNR and the code strings, were checked against earlier ones to keep backward compatibility. Browser tests open the real page with Playwright and check making codes, opening links, pasting and the address restriction.

Perceptual metrics such as SSIM track human judgment more closely [21], but PSNR was chosen as a simple and interpretable starting point. Fourteen photos are a small sample, and PSNR can disagree with perceived quality [21][22]. The 1200×1200 picture used for the large-limit tests is not included in the repository.

#### Design notes: the mathematics of PSNR

PSNR is defined as

```math
\mathrm{PSNR} = 10 \log_{10} \frac{255^2}{\mathrm{MSE}}
```

so a 10× larger MSE lowers PSNR by exactly 10 dB, and MSE = 0 gives infinity by definition. In the 4-pixel (red, yellow, blue, green) image, one channel off by 1 gives `MSE = 1/12` and therefore `PSNR = 10·log10(12 · 255²) ≈ 58.9 dB`.

The case-count chart starts from one observation. Every channel value of the 4-pixel image is 0 or 255, so if the other image's channel values range over 0–255, each channel's difference d is uniform over 0–255, and the distribution of the sum of squared errors is the 12-fold convolution of the distribution of d². The first version used FFT convolution [24], whose relative error of about 10⁻¹⁶ erased tail probabilities below 10⁻²⁰. In addition, `np.histogram` derives bin values from differences of a cumulative sum, which causes cancellation when tiny numbers are subtracted from values near 1 [23]. Only after switching to direct convolution, which adds positive terms only, and to `np.bincount` was the tail up to 58.9 dB obtained exactly.

The table of impressions for PSNR values is an empirical guide used in this project, not a standard from the literature. PSNR can disagree with quality as people perceive it [21][22].

### References

Sources for the concepts, standards and related work used in the design. Measurements come from the `lab/` scripts in [img-dev](https://github.com/Velvitty/img-dev) and the "Measurements" list below.

#### Papers and books

1. N. Ahmed, T. Natarajan, K. R. Rao, "Discrete Cosine Transform," *IEEE Transactions on Computers*, vol. C-23, no. 1, pp. 90–93, 1974. doi:10.1109/T-C.1974.223784
2. K. R. Rao, P. Yip, *Discrete Cosine Transform: Algorithms, Advantages, Applications*, Academic Press, 1990.
3. G. K. Wallace, "The JPEG still picture compression standard," *IEEE Transactions on Consumer Electronics*, vol. 38, no. 1, pp. xviii–xxxiv, 1992. doi:10.1109/30.125072
4. ITU-T Recommendation T.81 | ISO/IEC 10918-1, *Digital compression and coding of continuous-tone still images – Requirements and guidelines*, 1992.
5. E. Hamilton, *JPEG File Interchange Format, Version 1.02*, C-Cube Microsystems, 1992. (Standardized as ITU-T T.871.)
6. ITU-R Recommendation BT.601-7, *Studio encoding parameters of digital television for standard 4:3 and wide-screen 16:9 aspect ratios*, 2011.
7. C. E. Shannon, "A Mathematical Theory of Communication," *Bell System Technical Journal*, vol. 27, pp. 379–423, 623–656, 1948.
8. I. H. Witten, R. M. Neal, J. G. Cleary, "Arithmetic coding for data compression," *Communications of the ACM*, vol. 30, no. 6, pp. 520–540, 1987. doi:10.1145/214762.214771
9. G. N. N. Martin, "Range encoding: an algorithm for removing redundancy from a digitised message," Video & Data Recording Conference, Southampton, UK, 1979.
10. I. Pavlov, *LZMA SDK* (range coder: `LzmaEnc.c`, `LzmaDec.c`), https://www.7-zip.org/sdk.html
11. D. Marpe, H. Schwarz, T. Wiegand, "Context-based adaptive binary arithmetic coding in the H.264/AVC video compression standard," *IEEE Transactions on Circuits and Systems for Video Technology*, vol. 13, no. 7, pp. 620–636, 2003. doi:10.1109/TCSVT.2003.815173
12. J. Teuhola, "A compression method for clustered bit-vectors," *Information Processing Letters*, vol. 7, no. 6, pp. 308–311, 1978. doi:10.1016/0020-0190(78)90024-8 (Exp-Golomb codes)
13. S. W. Golomb, "Run-length encodings," *IEEE Transactions on Information Theory*, vol. 12, no. 3, pp. 399–401, 1966. doi:10.1109/TIT.1966.1053907
14. G. J. Sullivan, "Efficient scalar quantization of exponential and Laplacian random variables," *IEEE Transactions on Information Theory*, vol. 42, no. 5, pp. 1365–1374, 1996. doi:10.1109/18.532878
15. E. Y. Lam, J. W. Goodman, "A mathematical analysis of the DCT coefficient distributions for images," *IEEE Transactions on Image Processing*, vol. 9, no. 10, pp. 1661–1666, 2000. doi:10.1109/83.869177
16. J. L. Mannos, D. J. Sakrison, "The effects of a visual fidelity criterion on the encoding of images," *IEEE Transactions on Information Theory*, vol. 20, no. 4, pp. 525–536, 1974. doi:10.1109/TIT.1974.1055250
17. K. T. Mullen, "The contrast sensitivity of human colour vision to red-green and blue-yellow chromatic gratings," *The Journal of Physiology*, vol. 359, pp. 381–400, 1985. doi:10.1113/jphysiol.1985.sp015591
18. L. Williams, "Pyramidal parametrics," *Computer Graphics (SIGGRAPH '83)*, vol. 17, no. 3, pp. 1–11, 1983. doi:10.1145/964967.801126
19. J. W. Gibbs, "Fourier's Series," *Nature*, vol. 59, p. 606, 1899. doi:10.1038/059606a0; E. Hewitt, R. E. Hewitt, "The Gibbs-Wilbraham phenomenon: An episode in Fourier analysis," *Archive for History of Exact Sciences*, vol. 21, pp. 129–160, 1979. doi:10.1007/BF00330404
20. A. Ortega, K. Ramchandran, "Rate-distortion methods for image and video compression," *IEEE Signal Processing Magazine*, vol. 15, no. 6, pp. 23–50, 1998. doi:10.1109/79.733495
21. Z. Wang, A. C. Bovik, "Mean squared error: Love it or leave it? A new look at signal fidelity measures," *IEEE Signal Processing Magazine*, vol. 26, no. 1, pp. 98–117, 2009. doi:10.1109/MSP.2008.930649
22. Q. Huynh-Thu, M. Ghanbari, "Scope of validity of PSNR in image/video quality assessment," *Electronics Letters*, vol. 44, no. 13, pp. 800–801, 2008. doi:10.1049/el:20080522
23. D. Goldberg, "What every computer scientist should know about floating-point arithmetic," *ACM Computing Surveys*, vol. 23, no. 1, pp. 5–48, 1991. doi:10.1145/103162.103163
24. J. W. Cooley, J. W. Tukey, "An algorithm for the machine calculation of complex Fourier series," *Mathematics of Computation*, vol. 19, no. 90, pp. 297–301, 1965. doi:10.1090/S0025-5718-1965-0178586-1

#### Standards and specifications

25. S. Josefsson, *RFC 4648: The Base16, Base32, and Base64 Data Encodings*, IETF, 2006. (Section 5, base64url)
26. L. Masinter, *RFC 2397: The "data" URL scheme*, IETF, 1998.
27. T. Berners-Lee, R. Fielding, L. Masinter, *RFC 3986: Uniform Resource Identifier (URI): Generic Syntax*, IETF, 2005. (Query, fragment and percent-encoding)
28. R. Fielding, M. Nottingham, J. Reschke, *RFC 9110: HTTP Semantics*, IETF, 2022. (Section 15.5.15, 414 URI Too Long)
29. The Unicode Consortium, *The Unicode Standard*, Chapter 3, "Conjoining Jamo Behavior" (Hangul syllables U+AC00–U+D7A3), and *Unicode Standard Annex #15: Unicode Normalization Forms*. https://www.unicode.org/
30. WHATWG, *HTML Living Standard*, the canvas 2D context (`drawImage`, `imageSmoothingQuality`). https://html.spec.whatwg.org/multipage/canvas.html
31. Ecma International, *ECMA-262: ECMAScript Language Specification* (`Math.cos` is implementation-approximated). https://tc39.es/ecma262/

#### Technical documentation

32. GitHub Docs, "Managing a custom domain for your GitHub Pages site." https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site

#### Related work

33. Wolt, *BlurHash*. https://github.com/woltapp/blurhash
34. E. Wallace, *ThumbHash*, 2023. https://github.com/evanw/thumbhash

#### Data sources

35. Picsum Photos (photos provided by Unsplash). https://picsum.photos/ — img-dev `lab/imgs/` (ids 1015, 1025, 1062, 1084, 177, 237, 433, 64), img-dev `lab/val/` (ids 1003, 1035, 1074, 200, 287, 334)
36. C. E. Shannon, "Communication in the presence of noise," *Proceedings of the IRE*, vol. 37, no. 1, pp. 10–21, 1949. doi:10.1109/JRPROC.1949.232969 (sampling theorem)

#### Measurements

- DCInside comment limit of 400 characters, no display of images or `data:` links, and autolinking only with `https://`: checked in an actual comment box.
- GitHub Pages (g.yeon.at) 414 boundary: on 2026-10-26, requests with queries of 1,000–60,000 bytes returned 200 at 6,000 bytes and 414 at 9,000 bytes.
- Quality and speed figures: measured with the `lab/` scripts in [img-dev](https://github.com/Velvitty/img-dev) (Node.js 22). The 1200×1200 picture used for the large-limit tests is not in the repository.
