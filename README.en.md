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
| The page returns 404 | Check the repository's Pages settings, the `CNAME` file in the root and the DNS CNAME (`g` → `velvitty.github.io`). The first setup takes a few minutes to propagate. |

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

- **What:** A lossy image format that fits in one comment line (400 characters), and a browser page that creates and decodes it.
- **Why:** DCInside comments are limited to 400 characters, cannot carry attachments, and do not show `data:` links as images (confirmed by the user and measured in an earlier session). To send a photo, the photo itself has to be written out as text.
- **How (method and technique):** The classic lossy pipeline of transform coding (DCT) → quantization → context-adaptive binary arithmetic coding → text packing, redesigned for an extremely small budget of a few hundred bytes.
- **Mathematical approach:** One character carries `log2(alphabet size)` bits: `log2 64 = 6` for base64url and `log2 8192 = 13` for the 8,192 Hangul syllables used. After the 19-character address, 381 characters × 13 bits = 4,953 bits ≈ 619 bytes is all the photo gets. By Shannon's source coding theorem [7] the average code length cannot go below the model's entropy, so the design needs both a model that describes photos with low entropy (transform plus context model) and a coder that gets close to that entropy (arithmetic coding).
- **Engineering approach:** It must run from one static file with no server, and the code must be the entire data (self-contained). So there is no storage, no upload and no account.
- **Why this choice (alternatives):** (1) Links to an image host are short but depend on an upload and a server's lifetime, which conflicts with the no-upload requirement. (2) A JPEG `data:` link spends about 600 bytes on quantization and Huffman tables alone, over 800 characters in base64 [3][4][26]. (3) BlurHash [35] and ThumbHash [36] target placeholders of a few dozen characters with a fixed number of components (BlurHash allows at most 9 per axis), so they cannot use a 400-character budget. A dedicated codec whose quality grows with the budget was built instead.
- **Value:** No server cost or privacy risk, and as long as the text is copied, it decodes to the same photo anywhere, indefinitely.
- **Limitations:** The information ceiling is hard, so results cannot go beyond a blurry preview. Whether DCInside autolinks a link containing Hangul all the way through has not been verified.
- **Sources:** [3], [4], [7], [26], [35], [36]

#### Design notes: addresses and transport

- **What:** Links of the form `https://g.yeon.at/?code`, and the rule that switches to `#` for long codes.
- **Why:** DCInside comments only become clickable links when they start with `https://` (confirmed by the user). With the code after `?`, GitHub Pages ignores that part and serves the same `index.html`, so every address can open a different photo without any server setup.
- **How (method and technique):** On load the page reads `location.search`, or `location.hash` if there is no query, to get the code. On copy, if the URL-encoded address exceeds 7,000 bytes, `?` is replaced with `#`.
- **Mathematical approach:** A Hangul syllable is 3 bytes in UTF-8, and each byte becomes the 3 characters `%XX` in an address, so one syllable costs 9 bytes. A 381-character code is 381 × 9 = 3,429 bytes plus the 19-byte address. The 7,000-byte threshold corresponds to `(7000 − 19) / 9 ≈ 775` syllables.
- **Engineering approach:** In URI syntax [27] the query (after `?`) is sent to the server, while the fragment (after `#`) stays in the browser. HTTP allows a server to answer 414 to an overly long target [28]. Requests to g.yeon.at returned 200 at 6,000 bytes and 414 at 9,000 bytes (measured 2026-10-26), so the cut-off was set at 7,000 bytes to leave room for the rest of the request line.
- **Why this choice (alternatives):** Always using `#` would avoid 414 entirely, but the user wanted the `?` form, and short links have no problem with it. So short codes use `?` and long ones `#`.
- **Value:** 400-character links look exactly as the user wanted, and links of thousands of characters open without errors. The part after `#` never appears in server logs either.
- **Limitations:** The 414 boundary is a measurement of GitHub Pages (and its CDN) today and may change. How far a messenger's or forum's autolinker extends over non-ASCII characters differs by service.
- **Sources:** [27], [28], [32]

#### Design notes: static hosting

- **What:** Deploying only `index.html` and `CNAME` to GitHub Pages under the custom domain `g.yeon.at`.
- **Why:** Both compression and decoding happen in the browser, so the server only has to deliver a file.
- **How (method and technique):** `build.py` in img-dev produces `dist/`, which is copied into the img repository and pushed to `main`. DNS has a `g` CNAME → `velvitty.github.io`, and GitHub issues the certificate through Let's Encrypt [32][33].
- **Engineering approach:** Development material (img-dev) and the deployed copy (img) live in separate repositories so the deploy repository holds only what users should see. Because the account keeps its email private, commits must use the GitHub noreply address for pushes to be accepted [34].
- **Why this choice (alternatives):** A serverless function or own server could enforce the address restriction or collect usage statistics server-side, but would add cost and data-handling burden. Static hosting fits the requirements most simply.
- **Value:** No running cost, and GitHub Pages is the only point of failure.
- **Limitations:** Certificate issuance follows GitHub's schedule, so Enforce HTTPS cannot always be turned on right away. With no server code, checks such as the address restriction can only run in the browser.
- **Sources:** [32], [33], [34]

#### Design notes: preprocessing

- **What:** Shrinking the photo to the grid, compositing it on white, and converting it to YCbCr.
- **Why:** To size the transform to the budget, keeping computation low, and to separate brightness from color so color can be treated more coarsely.
- **How (method and technique):** On a canvas the photo is halved until it is 2–4 times the target, then drawn once more at grid size. The aspect ratio is stored in 6 bits as `a = round(16 · log2(width/height))`. Color conversion uses JFIF's full-range BT.601 coefficients (`Y = 0.299R + 0.587G + 0.114B`, and so on) [5][6].
- **Mathematical approach:** By the sampling theorem [38], shrinking by a factor k folds frequencies above `1/(2k)` of the original resolution back into the image (aliasing). The browser's bilinear filter looks at only a 2×2 neighborhood, so one large reduction is effectively point sampling. Halving step by step makes each step close to a 2×2 average, giving the same low-pass effect as a mipmap pyramid [18]. The aspect code has 16 steps per octave, so the ratio error is at most `2^(1/32) ≈ 2.2%`, and ±31 steps reach `2^(31/16) ≈ 3.83:1`.
- **Engineering approach:** A 64-pixel grid has 64 × 48 = 3,072 coefficients per channel: enough for the few hundred non-zero ones a 400-character budget (about 619 bytes) can hold, while the transform takes under a millisecond. The eye is less sensitive to color contrast than to brightness contrast [17], so separating the color channels lets quantization treat them differently.
- **Why this choice (alternatives):** Compressing RGB directly spends bits on information repeated in all three channels. Instead of 4:2:0-style spatial chroma subsampling, the color channels use a 1.7× larger step on the same grid. With a whole-image DCT, high-frequency color coefficients end up at zero anyway, so the effect is similar and the code is simpler.
- **Value:** Photos of any size go through the same path, and no staircase artifacts from downscaling leak into the result.
- **Limitations:** Canvas downscaling differs slightly between browsers, so the same photo can produce slightly different codes in different browsers. Photos longer than 3.8:1 are squeezed. Transparency is lost to white.
- **Sources:** [5], [6], [17], [18], [30], [38]

#### Design notes: transform

- **What:** A single 2D DCT-II over the whole grid, and the rule that orders coefficients by frequency.
- **Why:** Neighboring pixels in natural images are strongly correlated, so after a DCT the energy concentrates in a few low-frequency coefficients. The rest can then be set to zero with little visible loss.
- **How (method and technique):** The transform is computed separably, horizontally then vertically. Coefficients are sorted by `r = √((u·s/W)² + (v·s/H)²)` with `s = √(W·H)`.
- **Mathematical approach:** The normalization below makes `A[0,0]` the mean brightness and every other coefficient the amplitude of its cosine wave.

  ```math
  A[u,v] = \frac{\alpha_u \alpha_v}{WH} \sum_{x=0}^{W-1}\sum_{y=0}^{H-1} f(x,y)\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H},\quad \alpha_0=1,\ \alpha_{k>0}=2
  ```

  ```math
  f(x,y) = \sum_{u,v} A[u,v]\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H}
  ```

  The DCT is known to approach the optimal transform (KLT) for a first-order Markov process and to compact energy well [1][2]. Cost is `O(W·H·(W+H))`: about 340,000 multiplications for a 64 × 48 grid and about 270 million for 512 × 512.

- **Engineering approach:** Sorting by r, a frequency normalized to a square, puts the same visible frequency in the same position for wide and tall photos. The sorted order is cached per grid size, and ties on r are broken by the integers v and u, so browsers and Node.js produce the same order.
- **Why this choice (alternatives):** With JPEG-style 8×8 block DCTs [3][4], a small budget leaves only the DC of each block, producing a checkerboard. The reasons for blocks, computation and local adaptation, matter little at 64–512 grids, so a whole-image transform was chosen. Wavelets can keep edges sharper at the same budget but need a more complex implementation and context model.
- **Value:** Even with only low frequencies left, the image blurs smoothly without blocks: the right kind of degradation for a tiny budget.
- **Limitations:** Being global, fine texture in one area competes with the whole picture for the budget. A truncated cosine series overshoots by about 9% near edges (the Gibbs phenomenon) [19], causing ripples. No fast (FFT-based) DCT [24] is used yet, so a 512 grid takes close to a second.
- **Sources:** [1], [2], [3], [4], [19], [24]

#### Design notes: quantization

- **What:** The rule that turns real coefficients into integer levels: step size, dead zone, frequency and color weights, and the grid correction.
- **Why:** Most of the compression is decided here. Discarding less visible components more coarsely makes the result look better at the same length.
- **How (method and technique):** `step = 0.0002 · 1.06^q · (1 + 0.11·r) · (1.7 for color) · (64/long side in version 1)`, `level = sign(x) · ⌊|x|/step + 0.5 − 0.1⌋`, reconstructed as `level · step`.
- **Mathematical approach:** DCT AC coefficients are known to follow roughly a Laplacian distribution [15], and for Laplacian sources a uniform quantizer with a widened zero bin (dead zone) is favorable in rate-distortion terms [14]. Here a coefficient becomes zero below `0.6 · step`. q is exponential so that the wide range of `1.06^127 ≈ 1,620`× is divided evenly in 7 bits (6% per step). The version 1 correction follows from this: texture with per-pixel variance σ² has coefficient variance σ² under an orthonormal DCT, and the amplitude coefficients used here are the orthonormal ones times `2/√(WH)`. Their standard deviation therefore shrinks as `1/√(WH)`, which at a fixed aspect ratio is proportional to `1/long side`. Multiplying the step by `64/long side` keeps the same q meaning the same relative precision on every grid.
- **Engineering approach:** The frequency weight `(1 + 0.11r)` is a straight-line approximation of the contrast sensitivity function [16], and the 1.7× color factor reflects the lower contrast sensitivity for color [17]. Values were chosen by PSNR comparisons on the test photos (item 4 of [Experiments and decisions](https://github.com/Velvitty/img-dev/blob/main/README.en.md#experiments-and-decisions)).
- **Why this choice (alternatives):** JPEG-style 8×8 quantization tables are tied to the block size and do not fit a whole-image DCT or a changing grid. Pulling reconstruction values toward zero (shrink) is better in theory for Laplacian sources, but 0 measured best and was kept.
- **Value:** A single q adjusts the rate smoothly, which keeps fitting the character limit simple. The grid correction lets large budgets use all their characters.
- **Limitations:** The weights were chosen by PSNR on eight test photos and may not be optimal for perceived quality. In version 0, DC uses 6 bits for brightness and 5 for color, so the average color can be off by up to about 2/255 (brightness) and 4/255 (color).
- **Sources:** [14], [15], [16], [17]

#### Design notes: entropy coding

- **What:** The binary arithmetic coder (range coder) and context model that turn quantized integers into bits.
- **Why:** Most coefficients are zero, and the chance of being zero varies strongly with frequency and neighbors. Coding with those probabilities is what brings the length close to the entropy.
- **How (method and technique):** Each coefficient becomes a series of binary decisions: zero or not → sign → greater than 1 → greater than 2 → Exp-Golomb (the same binarization as CABAC [11][12]). Each decision is coded with a probability for its context (channel, frequency band, whether the previous two coefficients were zero). The coder follows the range coder structure of the LZMA SDK (32-bit range, carry handling through a cache byte) [9][10].
- **Mathematical approach:** An event of probability p costs `−log2 p` bits, and arithmetic coding makes the total length only a few bytes longer than that sum [8]; the spare bytes at the end are removed by the trailing-character trim in [Fitting the character limit](https://github.com/Velvitty/img-dev/blob/main/README.en.md#fitting-the-character-limit). Huffman coding spends at least 1 bit per symbol, even on a "zero" with probability 0.95, where arithmetic coding spends about 0.074 bits. Probabilities are 11-bit integers updated per bit as `p ← p + (2048 − p)/16` on 0 and `p ← p − p/16` on 1: an exponential moving average with an effective memory of about 16 bits, so it adapts quickly within a stream of only a few hundred bytes.
- **Engineering approach:** The model is kept small at 216 contexts in total. With little data, too many contexts would end before each one has learned anything. The header goes through the same coder at probability 0.5, costing exactly 1 bit per bit without separate bit-packing code.
- **Why this choice (alternatives):** SHIFT 4 (1/16) beat the LZMA default of 5 (1/32) by 0.04 dB on the validation photos. Starting probabilities trained on the test photos gained only 0.03–0.07 dB, too little for the cost of shipping a table in the code.
- **Value:** The same information is written far shorter than with fixed-length codes, so a 400-character code can hold hundreds of coefficients.
- **Limitations:** Signs still cost almost a full bit. Two-dimensional neighbors (the coefficient above) are not used as context, and no stronger model such as context mixing is used. One damaged bit corrupts everything after it.
- **Sources:** [8], [9], [10], [11], [12], [13]

#### Design notes: bitstream format

- **What:** The header holding version, grid, aspect ratio, quantizer step, average color and coefficient counts, followed by the coefficients.
- **Why:** The decoder must rebuild the same grid, coefficient order and step sizes from the code alone.
- **How (method and technique):** Version 0 has a fixed 62-bit header. Version 1 adds a 3-bit grid index after the version bit, uses 8 bits per DC value and `⌈log2(W·H)⌉` bits per coefficient count.
- **Mathematical approach:** 62 bits are 1.2% of a 400-character budget (about 5,200 bits). Storing coefficient counts in the header costs a fixed 10–16 bits per channel, instead of an end-of-block question on every coefficient.
- **Engineering approach:** The 64-pixel grid is still written as version 0, so after extending the format, existing codes and 400-character codes have not changed by a single bit (checked with 16 codes from the test photos). Grid index 7 is left unused as a marker for the next format extension.
- **Why this choice (alternatives):** A multi-bit version field from the start would have been tidier, but at 400 characters every bit counts, so it started as 1 bit and extensions use spare values inside version 1.
- **Value:** Overhead stays minimal for small budgets while leaving room to grow the format.
- **Limitations:** There is no checksum, so truncated or altered codes cannot be reliably detected and may decode into a wrong picture.
- **Sources:** [11]

#### Design notes: text packing

- **What:** Writing the bytes as Hangul syllables (13 bits) or base64url characters (6 bits).
- **Why:** Comments are limited by character count, so each character must carry as many bits as possible.
- **How (method and technique):** The bit string is cut into 13-bit groups written as `U+AC00 + value`, or 6-bit groups written in the base64url alphabet [25]. When reading, characters outside the ranges are ignored.
- **Mathematical approach:** Unicode has 11,172 Hangul syllables at `U+AC00–U+D7A3` [29]. Using all of them would give `log2 11172 ≈ 13.447` bits, 3.4% more than 13, but would require treating the whole code as one big number and converting its base, an `O(n²)` computation. Using a power of two, 8,192 characters, lets bits be cut and joined directly.
- **Engineering approach:** Spare bits in the last character are zero-filled, which matches the decoder's rule of reading missing bytes as zero and makes the trailing-character trim possible.
- **Why this choice (alternatives):** base83 (BlurHash [35]) or base85 carry slightly more than 6 bits but include address-special characters such as `#`, `%` and `?`. Emoji or Han characters could carry more bits, but display and input are unreliable. Hangul is natural for DCInside users and present in most fonts.
- **Value:** The same 400 characters hold about 2.2 times as much information as ASCII.
- **Limitations:** Where length is counted in UTF-8 bytes, Hangul carries 4.33 bits per byte, worse than ASCII's 6. A normalization that decomposes syllables into jamo (NFD) [29] breaks the code; the decoder does not currently convert pasted text back to NFC.
- **Sources:** [25], [29], [35]

#### Design notes: rate control and grid selection

- **What:** How the finest q and the most suitable grid size within the character limit are chosen.
- **Why:** Users only set a character count and expect the best quality within it.
- **How (method and technique):** q is binary-searched from 0 to 127 (7–8 encodes). Trailing characters are removed one at a time while the code still decodes identically. If the 64-pixel grid cannot fill the budget even at q = 0, the photo is re-encoded on the next grid, and the two results are compared by mean squared error against the original at the larger grid size; the better one is kept.
- **Mathematical approach:** The code length R(q) decreases almost monotonically with q, so binary search finds the smallest q with `R(q) ≤ budget` in about `⌈log2 128⌉ = 7` steps. Whether to spend a given rate on higher resolution with coarser quantization or lower resolution with finer quantization is a classic rate-distortion optimization problem [20]. Here it is solved with a greedy rule, trying a larger grid only once the smaller one is saturated, plus an actual error comparison.
- **Engineering approach:** A grid that fits at q = 0 skips the binary search, which cut a 7,000-character encode from about 1.8 s to 1.4 s. The trailing-character trim removes the spare bytes the range coder emits at the end, and since it checks by actually decoding, it cannot go wrong.
- **Why this choice (alternatives):** Encoding every grid and comparing all of them might find better results, but large grids take seconds each. A 768-pixel grid was tried and dropped because 20,000 characters took 16 seconds.
- **Value:** 400 characters stay as fast as before, while 7,000 characters use 6,968 of them and reach 28.0 dB.
- **Limitations:** R(q) is not strictly monotonic, so a few characters may be left unused. The greedy rule does not guarantee the optimal grid. Computation runs on the main thread, so the page pauses briefly for large budgets (no Web Worker).
- **Sources:** [8], [20]

#### Design notes: decoding and upscaling

- **What:** Computing the screen-size image directly from the coefficients.
- **Why:** Building a grid-size image and stretching it adds pixel steps or interpolation blur.
- **How (method and technique):** Screen coordinates, converted to grid units, are substituted for x and y in the inverse transform and the cosine series is evaluated directly. Rows are computed first, and rows whose coefficients are all zero are skipped.
- **Mathematical approach:** The inverse DCT formula is a smooth function defined for non-integer x as well, so sampling it at any resolution is band-limited interpolation in the cosine basis. Cost is `O(H·W·outW + rows·outH·outW)`.
- **Engineering approach:** The same function draws card previews (480px long side) and the View tab (960px). It ends with a YCbCr → RGB conversion clamped to 0–255.
- **Why this choice (alternatives):** An inverse DCT at grid size followed by canvas image scaling would be faster, but on small grids it shows the blurry cross patterns typical of bilinear interpolation.
- **Value:** Very little information still produces a pleasant, smoothly blurred image.
- **Limitations:** Gibbs ripples [19] stay visible when enlarged. `Math.cos` is implementation-approximated by specification [31], so pixel values may differ by about ±1 between browsers.
- **Sources:** [19], [31]

#### Design notes: page design

- **What:** The screens, code detection, address restriction and remembered settings inside one HTML file.
- **Why:** The user prefers one HTML file with inline JS and CSS that runs only in the browser, and recipients must be able to open it with no preparation.
- **How (method and technique):** The codec source is inserted into the page at build time. In pasted text, the longest run of Hangul (at least 4 characters) is taken as the code; failing that, the longest base64url run after the last `?` or `#`. Outside g.yeon.at, localhost, local files and the claude.ai preview, the page draws only a notice and stops.
- **Engineering approach:** Every localStorage access is wrapped so the page still works with defaults when storage is blocked. If the Clipboard API is refused, copying falls back to selecting the text. Canvas and image handling follow the HTML standard (an animated image draws its default frame) [30].
- **Why this choice (alternatives):** A framework would shorten the UI code but enlarge the file and add external dependencies. An address restriction is only reliable on a server, but static hosting has no server code.
- **Value:** Deployment is copying one file, and the page works offline (except fonts).
- **Limitations:** Code detection is a heuristic, so a Hangul sentence longer than a Hangul code can be mistaken for it. The address restriction is only a deterrent and does not stop someone who edits the source.
- **Sources:** [27], [30]

#### Design notes: build

- **What:** The script that combines the codec and page in `src/` into the deployable file and the preview.
- **Why:** Node.js tests and the browser must use the same codec file for the tested code to be the shipped code.
- **How (method and technique):** It removes only the `module.exports` line from `src/codec.js` and inserts the rest at `/*CODEC*/`.
- **Engineering approach:** Build outputs (`dist/`, `build/`) are committed too, so the deployable file can be used where no build tools exist.
- **Why this choice (alternatives):** Bundlers such as webpack are heavy for this scale and add runtime code to the output.
- **Value:** The build is reproduced by 30 lines of dependency-free Python.
- **Limitations:** Because outputs are committed, every change must be built and committed together so source and output do not drift.

#### Design notes: evaluation method

- **What:** How image quality is compared numerically (PSNR), plus the test photos and browser tests.
- **Why:** Choosing parameters, and checking that a change did not break earlier results, requires reproducible numbers.
- **How (method and technique):** The original box-downscaled to 160×120 is compared with the decoded result drawn at 160×120. Large-grid tests compare at 512×512. Test photos are 8 training and 6 validation images from picsum.photos (Unsplash photos) [37].
- **Mathematical approach:** The original's high frequencies cannot be transmitted anyway, so comparing near the size the recipient sees is what reveals differences between codecs. PSNR is a logarithm of mean squared error, simple to compute and easy to reproduce.
- **Engineering approach:** After every codec change, the 400-character results (average PSNR and the code strings) are checked against earlier ones to keep backward compatibility. Browser tests open the real page with Playwright to check making codes, opening links, pasting and the address restriction.
- **Why this choice (alternatives):** Perceptual metrics such as SSIM track human judgment better [21], but the work started with PSNR because it is simple and easy to interpret.
- **Value:** Whether a change improved quality or broke existing codes can be checked in seconds.
- **Limitations:** Fourteen photos is a small sample, and PSNR can disagree with perceived quality [21][22]. The test picture supplied by the user is not included in the repository.
- **Sources:** [21], [22], [37]

#### Design notes: the mathematics of PSNR

- **What:** The definition of PSNR and the case-count calculation for the 4-pixel example.
- **Why:** To answer questions such as "how many dB if identical?" exactly, and to give a scale for reading the numbers in this document.
- **How (method and technique):** Every channel value of the 4-pixel (red, yellow, blue, green) image is 0 or 255, so if the other image's channel values range over 0–255, each channel's difference d is uniform over 0–255. The distribution of the sum of squared errors is the 12-fold convolution of the distribution of d².
- **Mathematical approach:** `PSNR = 10·log10(255² / MSE)`, so a 10× larger MSE lowers PSNR by exactly 10 dB, and MSE = 0 gives infinity by definition. One channel off by 1 gives `MSE = 1/12` and `PSNR = 10·log10(12 · 255²) ≈ 58.9 dB`. In the case count, FFT convolution [24] has a relative error of about 10⁻¹⁶, which erased tail probabilities below 10⁻²⁰, and `np.histogram` derives bin values from differences of a cumulative sum, causing cancellation when subtracting tiny numbers from values near 1 [23]. Switching to direct convolution, which only adds positive terms, and `np.bincount` recovered the exact tail up to 58.9 dB.
- **Engineering approach:** The table of impressions for PSNR values is an empirical guide for this project, not a standard from the literature.
- **Value:** The meaning of the numbers is conveyed without misunderstanding, and the spurious bar caused by numerical error was corrected.
- **Limitations:** PSNR can disagree with quality as people perceive it [21][22].
- **Sources:** [21], [22], [23], [24]

### References

Sources for the concepts, standards and related work used in the design. The works below are cited from established knowledge; their original texts were not re-opened and checked during this work. Measurements come from the `lab/` scripts in [img-dev](https://github.com/Velvitty/img-dev) and the "Measurements and provided information" list below.

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
33. GitHub Docs, "Securing your GitHub Pages site with HTTPS." https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https
34. GitHub Docs, "Setting your commit email address." https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address

#### Related work

35. Wolt, *BlurHash*. https://github.com/woltapp/blurhash
36. E. Wallace, *ThumbHash*, 2023. https://github.com/evanw/thumbhash

#### Data sources

37. Picsum Photos (photos provided by Unsplash). https://picsum.photos/ — img-dev `lab/imgs/` (ids 1015, 1025, 1062, 1084, 177, 237, 433, 64), img-dev `lab/val/` (ids 1003, 1035, 1074, 200, 287, 334)
38. C. E. Shannon, "Communication in the presence of noise," *Proceedings of the IRE*, vol. 37, no. 1, pp. 10–21, 1949. doi:10.1109/JRPROC.1949.232969 (sampling theorem)

#### Measurements and provided information

- DCInside comment limit of 400 characters, no display of images or `data:` links, and autolinking only with `https://`: confirmed by the user and measured in an earlier session.
- GitHub Pages (g.yeon.at) 414 boundary: on 2026-10-26, requests with queries of 1,000–60,000 bytes returned 200 at 6,000 bytes and 414 at 9,000 bytes.
- g.yeon.at DNS (CNAME → velvitty.github.io, CAA allowing letsencrypt.org): looked up through dns.google on 2026-10-26.
- Quality and speed figures: measured with the `lab/` scripts in [img-dev](https://github.com/Velvitty/img-dev) (Node.js 22). The 1200×1200 picture used for the large-limit tests was supplied by the user and is not in the repository.
