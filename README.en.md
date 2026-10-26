<p align="center"><a href="README.md">한국어</a> · <a href="README.en.md">English</a> · <a href="README.ja.md">日本語</a> · <a href="README.es.md">Español</a></p>

# 400-Char Image Squeezer

A web page that turns a photo into text short enough for a single comment, and turns that text back into the photo.

**https://g.yeon.at/**

![The original and the results at 400, 1,000, 4,000 and 7,000 characters](docs/example.jpg)

<sub>From left: original, 400, 1,000, 4,000 and 7,000 characters (Hangul code, address included). Photo: Thomas Lefebvre / Unsplash</sub>

It was built for sharing photos where comments are short and images cannot be posted, such as DCInside. The photo is compressed in the browser to a few hundred bytes and becomes a single line like `https://g.yeon.at/?TqVTweO…`; the recipient opens the link to see the photo.

- No server, no upload, no storage. The code itself is the photo.
- Each Hangul syllable carries 13 bits, about 2.2 times as much as ASCII in the same number of characters.
- Raising the character limit grows the grid that holds the photo from 64 to 512 pixels, making it sharper.
- Everything runs from a single HTML file.

## Contents

1. [Usage](#usage)
2. [Codes and links](#codes-and-links)
3. [Quality](#quality)
4. [Privacy and requirements](#privacy-and-requirements)
5. [Limitations](#limitations)
6. [Troubleshooting](#troubleshooting)
7. [FAQ](#faq)
8. [Implementation notes](#implementation-notes)
   - [Design goals](#design-goals)
   - [Preprocessing](#preprocessing)
   - [Transform](#transform)
   - [Quantization](#quantization)
   - [Entropy coding](#entropy-coding)
   - [Bitstream](#bitstream)
   - [Text packing](#text-packing)
   - [Rate control and grid selection](#rate-control-and-grid-selection)
   - [Decoding and upscaling](#decoding-and-upscaling)
   - [Addresses and transport](#addresses-and-transport)
   - [Page design](#page-design)
   - [Quality evaluation](#quality-evaluation)
   - [The mathematics of PSNR](#the-mathematics-of-psnr)
9. [References](#references)

## Usage

The page has three tabs: **만들기** (Make), **보기** (View) and **JPG data 링크** (JPG data link).

### Make

1. Drop a photo, click the box to choose one, or paste with Ctrl/⌘+V. Several photos can be added at once.
2. Each photo gets a card: the original on the left, what the recipient will see on the right.
3. Press **복사** (Copy) to copy the address and code as one line, and paste that line into the comment as is.

Under each card are the original size, the code length, the total length against the limit, the grid size and the compression step (0–127, lower is finer).

| Setting | Default | Description |
|---|---|---|
| Code letters | ASCII | ASCII carries 6 bits per character, Hangul 13. DCInside comments only turn ASCII addresses into links, so ASCII is the default; Hangul is for sending the code alone for the recipient to paste into the View tab. |
| Character limit | 400 | Buttons for a DCInside comment (400), a tweet (280) and 1,000; any value can be typed in. |
| Address prefix | `https://g.yeon.at/?` | Text placed before the code. Leave it empty to output the code alone. |

Changing a setting re-compresses every photo on the page, and the settings are remembered in the browser.

### View

Opening a received link goes straight to this tab and shows the photo. A copied code can also be pasted here; surrounding text is ignored and only the code is read.

### JPG data link

A tool for places that show `data:` links as images (sites that allow HTML or Markdown images). It re-encodes the photo as JPG and produces a `data:image/jpeg;base64,…` link, either searching for the best quality within a target length or using a chosen quality, and copies it as a link, an `<img>` tag or Markdown. At the same character count the code from the Make tab is sharper (see [Quality](#quality)).

## Codes and links

| Format | Example | Characters for the code | Capacity |
|---|---|---|---|
| ASCII + address (default) | `https://g.yeon.at/?TqVTweO…` | 381 | about 285 bytes |
| Hangul + address | `https://g.yeon.at/?뗈앏괟…` | 381 | about 619 bytes |
| Hangul code only | `뗈앏괟…` | 400 | 650 bytes |
| ASCII code only | `TqVTweO…` | 400 | 300 bytes |

- The address must start with `https://` to become a clickable link in comments. DCInside comments only link addresses written in ASCII, so an address with a Hangul code is not clickable.
- When the code is long enough that the address would pass about 7 KB (ASCII codes of roughly 6,980 characters or Hangul codes of roughly 770 or more), `#` is used instead of `?`. `https://g.yeon.at/#code` opens the same photo, but DCInside comments leave the part after `#` out of the link.
- `https://g.yeon.at/index.html?code` also works but uses 10 more characters.

## Quality

Raising the character limit grows the grid that holds the photo and makes it sharper. The bear photo from the example image (1280×960), compressed as Hangul code and measured at 640×480:

| Limit | Characters used | Grid | PSNR | JPG `data:` link of the same length |
|---|---|---|---|---|
| 400 | 387 | 64px | 23.9 dB | does not fit |
| 1,000 | 955 | 96px | 25.5 dB | does not fit |
| 4,000 | 3,810 | 256px | 27.7 dB | 26.2 dB |
| 7,000 | 6,877 | 512px | 28.8 dB | 27.4 dB |
| 10,000 | 8,526 | 512px | 29.2 dB | 28.2 dB |

The grid tops out at 512 pixels, so for this photo nothing changes beyond about 8,500 characters. Compression takes under 0.1 s up to 1,000 characters, about 0.4 s at 4,000 and about 3 s from 7,000.

At 400 characters (address included), eight photos measured at 160×120:

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

- Photos built from large shapes and smooth color changes (fog, sky, close-ups of faces or animals) come out well.
- Photos covered in fine texture, such as wood grain, grass or rock, are blurrier at the same length.
- Cropping to the part worth showing makes the result much sharper at the same length.
- Text cannot be read at 400 characters.

PSNR measures how close a result is to the original; higher is closer. The guide below is the empirical scale used in this document (see [The mathematics of PSNR](#the-mathematics-of-psnr)).

| PSNR | Impression |
|---|---|
| ∞ | Identical to the original |
| 45 dB and up | Practically indistinguishable |
| 35–40 dB | Typical JPG |
| Around 30 dB | Smearing visible on close inspection |
| Around 25 dB | Blurry but recognizable |
| Around 20 dB | Only shapes and colors remain |

## Privacy and requirements

- Photos never leave the browser. Compression and decoding both run in the page's own script.
- Nothing is stored on a server, so anyone with the code can see the same photo. Delete the code and the photo is gone.
- The only external resource is the fonts (Google Fonts); without them, default fonts are used.
- Browser storage (localStorage) keeps only the code letters, the address prefix and the last tab. Photos and codes are not stored.
- Works in recent Chrome, Edge, Safari and Firefox, including on phones.
- The page only runs on g.yeon.at. Copied to another address, it shows a notice and a button to g.yeon.at, passing along any code in the address.

## Limitations

- At 400 characters the result is a blurry preview; text and fine texture cannot be made out.
- Ripples appear near sharp brightness edges.
- The grid tops out at 512 pixels, so beyond a point more characters stop helping.
- Limits of 7,000 characters or more take a few seconds to compress, and the page pauses meanwhile.
- Photos longer than about 3.8:1 are squeezed to that ratio, transparent areas become white, and animated GIFs keep only their first frame.
- Where length is counted in UTF-8 bytes, Hangul codes are at a disadvantage; use the ASCII code there.
- In DCInside comments, Hangul codes and anything after `#` are not included in links. A Hangul code is less than half as long, but the recipient has to copy it and paste it into the View tab.

## Troubleshooting

| Symptom | Fix |
|---|---|
| "이 글자로는 사진을 만들 수 없습니다" (cannot make a photo from this text) | The code was cut off or other characters got mixed in. Copy the whole line again. |
| The link opens the Make tab with no photo | The comment's autolink stopped before `?` or `#`, which happens with Hangul codes and `#` links. Copy the code from the comment and paste it into the View tab. Senders can use the ASCII code. |
| The copy button does nothing | The text is already selected; press Ctrl/⌘+C. |
| The photo looks nothing like the original | The code may come from a version with a different compression scheme. |
| The total is a little under the limit | This is normal. Compression steps come in jumps, so a few characters can be left over. |

## FAQ

**Can the original photo be recovered from a code?**
No. A code only holds as much information as a blurry preview.

**Does the same photo always give the same code?**
In the same browser, yes. Browsers shrink images slightly differently, so codes can differ a little between browsers, but once made, a code decodes to the same photo in any browser.

**Why is the Hangul code sharper than the ASCII code?**
The same number of characters holds about 2.2 times as much information. An ASCII character is one of 64 values; a Hangul character is one of 8,192.

**The Hangul code looks like strange characters.**
It includes rarely used syllables such as `뗈` and `쯿`. The characters carry no meaning on purpose, and most fonts include all of them.

## Implementation notes

Compression runs in the order below. This chapter explains why each stage was built the way it is, which mathematical and engineering arguments it rests on, and what was gained and given up.

```
photo → shrink to grid (64–512px) → YCbCr → whole-image 2D DCT → quantization → context-adaptive binary arithmetic coding → Hangul or ASCII text
```

### Design goals

A DCInside comment cannot exceed 400 characters, cannot carry an attached image, and does not show `data:` links as pictures. Under these conditions the only way to pass on a photo is to write the photo itself as text. So a lossy format that fits in one comment line was designed together with a browser page that creates and decodes it. The compression follows the classic chain of transform coding, quantization, entropy coding and text packing, but every stage is re-tuned for a budget of only a few hundred bytes.

The budget starts with how much one character can carry: `log2(alphabet size)` bits. base64url carries `log2 64 = 6` bits, and the 8,192 Hangul syllables used here carry `log2 8192 = 13` bits. The 381 characters left after the 19-character address, at 13 bits each, give 4,953 bits, about 619 bytes, for the whole photo. By Shannon's source coding theorem [1] the average code length cannot fall below the model's entropy. A recognizable picture fits in this budget only with both a model that describes photos with low entropy (a transform and a context model) and a coder that comes close to that entropy (arithmetic coding).

Other routes were weighed. A link to an image host is short but depends on an upload and on a server's lifetime, which conflicts with the goal of a code that is complete on its own. A JPEG `data:` link [2] spends about 600 bytes on quantization and Huffman tables alone, more than 800 characters in base64 [3][4]. BlurHash [5] and ThumbHash [6] are built for placeholders of a few dozen characters with a fixed number of components (BlurHash allows at most 9 per axis), so they cannot use a 400-character budget. Hence a dedicated codec whose quality keeps growing as the budget grows.

The design brings no server cost and no privacy risk, and as long as the text is copied it decodes to the same photo anywhere. In return the information ceiling is hard: at 400 characters the result cannot go beyond a blurry preview.

### Preprocessing

Preprocessing shrinks the photo to the grid size, composites it on white and converts it to YCbCr. Sizing the transform to the budget keeps computation low, and separating brightness from color lets color be treated more coarsely. Color conversion uses JFIF's full-range BT.601 coefficients (`Y = 0.299R + 0.587G + 0.114B` and so on) [7][8]. Transparent areas are filled with white.

Downscaling is bound by sampling theory. By the sampling theorem [9], shrinking by a factor k folds frequencies above `1/(2k)` of the original resolution back into the image as aliasing. The browser's bilinear filter only looks at a 2×2 neighborhood, so one large reduction is effectively point sampling and cannot avoid the problem. The photo is therefore halved on a canvas until it is 2–4 times the target and then drawn once more at grid size [10]. Each step comes close to a 2×2 average, giving the same low-pass effect as a mipmap pyramid [11].

The aspect ratio is stored in 6 bits as `a = round(16 · log2(width/height))`. With 16 steps per octave the ratio error is at most `2^(1/32) ≈ 2.2%`, and ±31 steps cover up to about 3.83:1. A 64-pixel grid has 64 × 48 = 3,072 coefficients per channel, enough for the few hundred non-zero ones a 400-character budget can hold, while the transform finishes within a millisecond.

Color is kept separate because the eye is less sensitive to color contrast than to brightness contrast [12]. Compressing RGB directly wastes bits on information repeated across the three channels. Instead of reducing color spatially as in 4:2:0, the color channels use a 1.7× larger quantization step on the same grid. With a whole-image DCT the high-frequency color coefficients end up at zero anyway, so the effect is similar and the implementation is simpler.

Photos of any size thus follow the same path, and no staircase artifacts from downscaling leak into the result. Canvas downscaling does differ slightly between browsers, so the same photo can yield slightly different codes in different browsers; photos longer than 3.8:1 are squeezed to that ratio, and transparency is lost to white.

### Transform

Neighboring pixels in natural images are strongly correlated. After a DCT the energy gathers in a few low-frequency coefficients, and the rest can be set to zero with little visible loss. The DCT is known to approach the optimal transform (KLT) for a first-order Markov process and to compact energy well [13][14]. This codec applies one 2D DCT-II to the whole grid with the following normalization.

```math
A[u,v] = \frac{\alpha_u \alpha_v}{WH} \sum_{x=0}^{W-1}\sum_{y=0}^{H-1} f(x,y)\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H},\quad \alpha_0=1,\ \alpha_{k>0}=2
```

```math
f(x,y) = \sum_{u,v} A[u,v]\cos\frac{\pi(2x+1)u}{2W}\cos\frac{\pi(2y+1)v}{2H}
```

Under this normalization `A[0,0]` is the mean brightness and every other coefficient is the amplitude of its cosine wave, so quantization steps can be set directly in brightness units. The transform is computed separably, horizontally then vertically, at a cost of `O(W·H·(W+H))`: about 340,000 multiplications for a 64 × 48 grid and about 270 million for 512 × 512.

Coefficients are ordered by increasing `r = √((u·s/W)² + (v·s/H)²)` with `s = √(W·H)`. Using a frequency normalized to a square puts the same visible frequency in the same position for wide and tall photos. The order is cached per grid size, and ties on r are broken by the integers v and u so that browsers and Node.js produce identical orders.

There is a reason for not splitting into 8×8 blocks as JPEG does [3]. With a small budget only the DC of each block survives and a checkerboard appears. The original motivations for blocks, computation and local adaptation, matter little at 64–512 grids. Wavelets can preserve edges better at the same budget but make the implementation and context model more complex. A whole-image DCT blurs smoothly without blocks even when only low frequencies remain, which is the right kind of degradation for a tiny budget.

The price is that, being global, fine texture in one area competes with the whole picture for the budget. A truncated cosine series overshoots by about 9% near edges (the Gibbs phenomenon) [15] and leaves ripples. No fast FFT-based DCT [16] is used yet, so the transform alone takes close to a second on a 512 grid.

### Quantization

Most of the compression is decided in quantization: the more coarsely the less visible components are discarded, the better the result looks at the same length. The step size and rounding rule are:

```math
\mathrm{step} = 0.0002 \cdot 1.06^{q} \cdot (1 + 0.11\,r) \cdot c_{\mathrm{chroma}} \cdot s_{\mathrm{grid}},\qquad \mathrm{level} = \mathrm{sign}(x)\left\lfloor \frac{|x|}{\mathrm{step}} + 0.5 - 0.1 \right\rfloor
```

Here `c = 1.7` for color channels and `s = 64/long side` for grids of 96 pixels or more; reconstruction is `level · step`.

DCT AC coefficients are known to follow roughly a Laplacian distribution [17], and for Laplacian sources a uniform quantizer with a widened zero bin (dead zone) is favorable in rate-distortion terms [18]. In the rule above a coefficient becomes zero below `0.6 · step`. The frequency weight `1 + 0.11r` is a straight-line approximation of the contrast sensitivity function [19], and the 1.7× color factor reflects the lower sensitivity to color contrast [12].

q is exponential so that a range of `1.06^127 ≈ 1,620`× is divided evenly in 7 bits. Each step changes the step size by 6%, so the rate can be adjusted smoothly, which keeps fitting the character limit simple. The first version used a base of 0.004, a factor of 1.09 and a 6-bit q; it was so coarse that even the finest step used only 110–190 of 360 characters, so the range was widened to the current one.

The weights were chosen by changing one value at a time and comparing average PSNR over the eight tuning photos (averaging ASCII 360, Hangul 360 and ASCII 150 characters). The baseline (beta 0.11, chroma 1.7, dz 0.22, shrink 0.08) averaged 23.165 dB.

| Parameter | Value and average PSNR |
|---|---|
| beta (frequency weight) | 0.05 → 23.183, 0.08 → 23.180, 0.15 → 23.152, 0.2 → 23.106 |
| chroma (color factor) | 1.3 → 23.125, 2.2 → 23.153, 3 → 23.119 |
| dz (dead zone) | 0.1 → 23.253, 0.3 → 23.076, 0.4 → 22.965 |
| shrink (pull toward zero) | 0 → 23.213, 0.15 → 23.102 |

dz and shrink, with clear differences, became 0.1 and 0; beta and chroma, within 0.02 dB, were kept. Pulling reconstruction values toward zero is better in theory for Laplacian sources, but leaving them unshifted measured best.

The grid factor `s = 64/long side` follows from this relation. Texture with per-pixel variance σ² has coefficient variance σ² under an orthonormal DCT. The amplitude coefficients used here are the orthonormal ones multiplied by `2/√(WH)`, so their standard deviation shrinks as `1/√(WH)`, which at a fixed aspect ratio is proportional to `1/long side`. Multiplying the step by `64/long side` makes the same q mean the same relative precision on every grid. Without this factor fine texture is still rounded to zero on large grids: the bear photo from the example image stopped at 1,303 characters (26.1 dB) even on the 512-pixel grid, while with the factor a 7,000-character limit uses 6,877 characters and reaches 28.8 dB.

JPEG-style 8×8 quantization tables are tied to the block size and do not suit a whole-image DCT or a changing grid. The weights were chosen by PSNR on the test photos and may differ from the optimum for perceived quality. On the 64-pixel grid the DC uses 6 bits for brightness and 5 for color, so the average color can be off by up to about 2/255 (brightness) and 4/255 (color); grids of 96 pixels or more raise this to 8 bits.

### Entropy coding

Most quantized coefficients are zero, and the chance of a coefficient being zero varies strongly with its frequency and neighbors. Only when that variation is reflected in the probabilities does the length come close to the entropy. The integers are therefore not written out directly but coded with a binary arithmetic coder that uses context-dependent probabilities.

An event of probability p costs `−log2 p` bits, and arithmetic coding makes the total only a few bytes longer than that sum [20]. Huffman coding spends at least one bit per symbol, even on a "zero" with probability 0.95, where arithmetic coding spends about 0.074 bits. The spare bytes left at the end are removed by the trailing-character trim described in [Rate control and grid selection](#rate-control-and-grid-selection).

Each coefficient is split into binary decisions in the order zero or not, sign, greater than 1, greater than 2, and Exp-Golomb, the same binarization as CABAC in H.264 [21][22][23]. Each decision is coded with the probability of a context defined by the channel, the frequency band and whether the previous two coefficients were zero. The coder is a range coder [24] implemented after the structure of the LZMA SDK, with a 32-bit range and carry handling through a cache byte [25].

Probabilities are 11-bit integers updated per bit as `p ← p + (2048 − p)/16` on 0 and `p ← p − p/16` on 1. This exponential moving average has an effective memory of about 16 bits, so it adapts quickly even within a stream of a few hundred bytes. On the six validation photos, update rates of 1/8, 1/16 and 1/32 averaged 25.372, 25.385 and 25.344 dB, so 1/16 was chosen. The model is kept small at 216 contexts, because with little data too many contexts would end the stream before each had learned anything. Starting probabilities trained on the test photos gained only 0.03–0.07 dB, too little to justify shipping a table in the code. The header goes through the same coder at probability 0.5 and costs exactly one bit per bit with no separate bit-packing code.

Written this way, the same information takes far fewer bits than with fixed-length codes, so a 400-character code holds hundreds of coefficients. Weaknesses remain: signs still cost almost a full bit, two-dimensional neighbors such as the coefficient above are not used as context, no stronger model such as context mixing is used, and damage in the middle of a stream corrupts everything after it.

### Bitstream

The decoder must rebuild the same grid, coefficient order and step sizes from the code alone. The code therefore starts with a header, followed by the Y, Cb and Cr coefficients. The header goes through the same coder at probability 0.5. The 64-pixel grid is written in format 0.

| Field | Bits | Value |
|---|---|---|
| Format | 1 | 0 |
| Aspect code | 6 | a + 32 |
| Compression step q | 7 | 0–127 |
| Y DC | 6 | mean brightness × 63 |
| Cb, Cr DC | 5, 5 | (mean + 0.5) × 31 |
| Y, Cb, Cr coefficient counts | 12, 10, 10 | 0–4,095, 0–1,023 |

Grids of 96 pixels or more are written in format 1: after the format bit comes a 3-bit grid index (0–6 for 64, 96, 128, 192, 256, 384 and 512 pixels), DC values take 8 bits each and coefficient counts `⌈log2(W·H)⌉` bits each.

The format 0 header is 62 bits, 1.2% of a 400-character budget of about 5,200 bits. Storing the coefficient counts in the header instead of an end-of-block marker costs a fixed 10–16 bits per channel, with no per-coefficient question of whether the block ends here [21]. When the growing-grid format was added, the 64-pixel grid stayed in format 0, so codes already in circulation and 400-character codes did not change by a single bit (checked with 16 codes from the test photos). A multi-bit format field from the start would have been tidier, but at 400 characters every bit counts, so it started as one bit. Both values are now in use, so the next extension will use the unused grid index 7 as a marker. The remaining weakness is the lack of a checksum: truncated or altered codes cannot be reliably detected and may decode into a wrong picture.

### Text packing

Comments are limited by character count, so each character must carry as many bits as possible. The bytes are joined into a bit string and cut into 13-bit groups written as the Hangul syllable `U+AC00 + value`, or into 6-bit groups written as base64url characters [26]. Characters outside these ranges are ignored when reading.

Unicode has 11,172 Hangul syllables at `U+AC00–U+D7A3` [27]. Using all of them would carry `log2 11172 ≈ 13.447` bits, 3.4% more than 13. That, however, would require treating the whole code as one big number and converting its base, an `O(n²)` computation, whereas with a power of two, 8,192 characters, bits can simply be cut and joined. Spare bits in the last character are zero-filled, which matches the decoder's rule of reading missing bytes as zero and makes the trailing-character trim possible.

Other alphabets were considered. base83 (BlurHash [5]) or base85 carry slightly more than 6 bits but include address-special characters such as `#`, `%` and `?`. Emoji or Han characters could carry more bits, but their display and input are unreliable. Hangul is natural in DCInside comments, present in most fonts, and holds about 2.2 times as much information as ASCII in the same 400 characters.

Where length is counted in UTF-8 bytes, however, Hangul carries 4.33 bits per byte, worse than ASCII's 6. A normalization that decomposes syllables into jamo (NFD) [27] also breaks a code, and the decoder does not currently convert pasted text back to NFC. And since DCInside comments only autolink addresses written in ASCII, a Hangul code can be passed on only by copying and pasting, not as a link. That is why the page's default code letters are ASCII.

### Rate control and grid selection

Only a character limit is given, and within it the quality should be as high as possible. To achieve this, the compression step q and the grid size are chosen together.

The code length R(q) decreases almost monotonically with q, so a binary search over 0–127 finds the smallest q with `R(q) ≤ budget` in about `⌈log2 128⌉ = 7` encodes. Trailing characters are then removed one at a time for as long as the decoded result stays the same. This removes the spare bytes the range coder emits at the end, and since it is checked by actually decoding, it cannot go wrong [20].

Whether a given rate is better spent on resolution or on quantization precision is a classic rate-distortion optimization problem [28]. It is solved here with a greedy rule. The photo is first encoded on the 64-pixel grid. If even q = 0 leaves characters unused, that grid cannot fill the budget, so the photo is re-encoded on the next grid size. Both results are compared by mean squared error against the original at the larger grid size and the better one is kept; the search stops when the larger grid is worse or no longer reaches q = 0. A grid that fits at q = 0 skips the binary search, which cut a 7,000-character encode of the bear photo from 3.5 s to 2.9 s.

Encoding every grid and comparing them all might find better results, but large grids take seconds each. A 768-pixel grid was tried as well: compressing the bear photo with a 20,000-character limit used 13,003 characters and reached 29.9 dB, but took 7.5 s. That is only 0.7 dB above the best the 512-pixel grid can do (8,526 characters, 29.2 dB, 3.2 s), so the grid stops at 512 pixels. The outcome is that a 400-character encode finishes in 0.01 s, while a 7,000-character limit uses 6,877 characters and reaches 28.8 dB (Node.js 22). R(q) is not strictly monotonic, so a few characters may be left over; the greedy rule does not guarantee the optimal grid; and the computation runs on the main thread, so the page pauses briefly for large budgets.

### Decoding and upscaling

Building a grid-size image and then stretching it would add pixel steps or interpolation blur, so the decoder computes the screen-size image directly from the coefficients. Screen coordinates, converted to grid units, are substituted for x and y in the inverse transform and the cosine series is summed as is. The inverse DCT is a smooth function defined for non-integer x as well, so sampling it at any resolution amounts to band-limited interpolation in the cosine basis.

Rows are computed first, and rows whose coefficients are all zero are skipped, for a cost of `O(H·W·outW + rows·outH·outW)`. The same function draws card previews (480px long side) and the View tab (960px), ending with a YCbCr-to-RGB conversion clamped to 0–255. An inverse DCT at grid size followed by canvas scaling would be faster, but on small grids it shows the blurry cross patterns typical of bilinear interpolation.

This approach turns very little information into a pleasant, smoothly blurred image. Gibbs ripples [15] remain visible when enlarged, and because `Math.cos` is implementation-approximated by specification [29], pixel values may differ by about ±1 between browsers.

### Addresses and transport

The default link has the form `https://g.yeon.at/?code`. DCInside only turns ASCII addresses starting with `https://` into clickable links, and static hosting ignores everything after `?` and serves the same `index.html`. As a result every address can open a different photo with no server configuration. On load the page reads `location.search`, or `location.hash` when the query is empty, to get the code.

In URI syntax [30] the query after `?` is sent to the server while the fragment after `#` stays in the browser. A Hangul syllable is 3 bytes in UTF-8, and each byte becomes the three characters `%XX` in an address, so one syllable takes 9 bytes. A 381-character code therefore takes 381 × 9 = 3,429 bytes plus the 19-byte address. An HTTP server may answer 414 to an overly long target [31], and direct requests to g.yeon.at were accepted at 6,000 bytes and rejected with 414 at 9,000. To leave room for the rest of the request line, Copy replaces `?` with `#` once the URL-encoded address passes 7,000 bytes, which corresponds to `(7000 − 19) / 9 ≈ 775` syllables.

Using `#` all the time would remove any worry about 414, but `?` is the default form and is enough for short links. So 400-character links keep the `?` form, while links of thousands of characters switch to `#` and open without errors; the part after `#` never reaches server logs either. The 414 boundary is a measurement of today's hosting and may change, and since DCInside comments leave Hangul and anything after `#` out of links, links of thousands of characters that switched to `#` and Hangul codes can only be passed on there by pasting.

### Page design

Compression and decoding both happen in the browser, so the server only has to deliver a file. Static hosting on GitHub Pages under the custom domain `g.yeon.at` was therefore chosen [32]. A serverless function or an own server could handle the address restriction or usage statistics on the server side, at the price of running costs and the burden of handling data; static hosting has no running cost and a single point of failure.

The page is one HTML file with all JS and CSS inline. Recipients must be able to open it with no preparation, and deployment should amount to copying one file. The codec is inserted into the page at build time, and apart from fonts the page works offline. A framework would shorten the interface code but enlarge the file and add external dependencies.

To find the code in pasted text, the longest run of at least four Hangul characters is chosen, or failing that, the longest base64url run after the last `?` or `#` [30]. The code is read even when other words surround it, but a Hangul sentence longer than the code can be picked by mistake. Every localStorage access is guarded so the page falls back to defaults when storage is blocked, and if the clipboard API is refused, copying falls back to selecting the text. Canvas and image handling follow the HTML standard, under which an animated image draws its default frame [10].

The address restriction makes the page draw only a notice and stop when it is opened outside g.yeon.at, localhost, local files and the preview environment. A reliable restriction belongs on a server, but static hosting has no server code, so this check remains a deterrent that does not stop someone who edits the source.

### Quality evaluation

Choosing parameters, and checking that a change did not break earlier results, requires reproducible numbers. The original is box-downscaled and compared by PSNR with the decoded result drawn at the same size. The 400-character results were measured at 160×120, close to the size the recipient sees, and the effect of the character limit at 640×480. The original's high frequencies cannot be carried by 400 characters anyway, so only a comparison near the viewing size reveals the differences between codecs. PSNR, a logarithm of mean squared error, is simple to compute and easy to reproduce.

The test photos are Unsplash photos from picsum.photos: eight used for tuning parameters and six for validating results [33]. After every codec change, the 400-character results, both average PSNR and the code strings, were checked against earlier ones to keep backward compatibility. For the comparison with JPG `data:` links, Pillow (libjpeg) was used to try different sizes and qualities and keep the best result that fits the character count. Browser tests open the real page with Playwright and check making codes, opening links, pasting and the address restriction.

Perceptual metrics such as SSIM track human judgment more closely [34], but PSNR was chosen as a simple and interpretable starting point. Fourteen photos are a small sample, and PSNR can disagree with perceived quality [34][35].

### The mathematics of PSNR

PSNR is defined as

```math
\mathrm{PSNR} = 10 \log_{10} \frac{255^2}{\mathrm{MSE}}
```

so a 10× larger MSE lowers PSNR by exactly 10 dB, and MSE = 0 gives infinity by definition. In a 4-pixel (red, yellow, blue, green) image, one channel off by 1 gives `MSE = 1/12` and therefore `PSNR = 10·log10(12 · 255²) ≈ 58.9 dB`.

The distribution of PSNR over every possible image starts from one observation. Every channel value of the 4-pixel image is 0 or 255, so if the other image's channel values range over 0–255, each channel's difference d is uniform over 0–255, and the distribution of the sum of squared errors is the 12-fold convolution of the distribution of d². The first version used FFT convolution [16], whose relative error of about 10⁻¹⁶ erased tail probabilities below 10⁻²⁰. In addition, `np.histogram` derives bin values from differences of a cumulative sum, which causes cancellation when tiny numbers are subtracted from values near 1 [36]. Only after switching to direct convolution, which adds positive terms only, and to `np.bincount` was the tail up to 58.9 dB obtained exactly. The median PSNR of a randomly colored image turns out to be 4.8 dB, and an identical image is one case in about 7.9 × 10²⁸.

The PSNR impression table in [Quality](#quality) is an empirical guide used in this project, not a standard from the literature. PSNR can disagree with quality as people perceive it [35].

## References

1. C. E. Shannon, "A Mathematical Theory of Communication," *Bell System Technical Journal*, vol. 27, pp. 379–423, 623–656, 1948.
2. L. Masinter, *RFC 2397: The "data" URL scheme*, IETF, 1998.
3. G. K. Wallace, "The JPEG still picture compression standard," *IEEE Transactions on Consumer Electronics*, vol. 38, no. 1, pp. xviii–xxxiv, 1992. doi:10.1109/30.125072
4. ITU-T Recommendation T.81 | ISO/IEC 10918-1, *Digital compression and coding of continuous-tone still images – Requirements and guidelines*, 1992.
5. Wolt, *BlurHash*. https://github.com/woltapp/blurhash
6. E. Wallace, *ThumbHash*, 2023. https://github.com/evanw/thumbhash
7. E. Hamilton, *JPEG File Interchange Format, Version 1.02*, C-Cube Microsystems, 1992. (standardized as ITU-T T.871)
8. ITU-R Recommendation BT.601-7, *Studio encoding parameters of digital television for standard 4:3 and wide-screen 16:9 aspect ratios*, 2011.
9. C. E. Shannon, "Communication in the presence of noise," *Proceedings of the IRE*, vol. 37, no. 1, pp. 10–21, 1949. doi:10.1109/JRPROC.1949.232969 (sampling theorem)
10. WHATWG, *HTML Living Standard*, the canvas 2D context (`drawImage`, `imageSmoothingQuality`). https://html.spec.whatwg.org/multipage/canvas.html
11. L. Williams, "Pyramidal parametrics," *Computer Graphics (SIGGRAPH '83)*, vol. 17, no. 3, pp. 1–11, 1983. doi:10.1145/964967.801126
12. K. T. Mullen, "The contrast sensitivity of human colour vision to red-green and blue-yellow chromatic gratings," *The Journal of Physiology*, vol. 359, pp. 381–400, 1985. doi:10.1113/jphysiol.1985.sp015591
13. N. Ahmed, T. Natarajan, K. R. Rao, "Discrete Cosine Transform," *IEEE Transactions on Computers*, vol. C-23, no. 1, pp. 90–93, 1974. doi:10.1109/T-C.1974.223784
14. K. R. Rao, P. Yip, *Discrete Cosine Transform: Algorithms, Advantages, Applications*, Academic Press, 1990.
15. J. W. Gibbs, "Fourier's Series," *Nature*, vol. 59, p. 606, 1899. doi:10.1038/059606a0; E. Hewitt, R. E. Hewitt, "The Gibbs-Wilbraham phenomenon: An episode in Fourier analysis," *Archive for History of Exact Sciences*, vol. 21, pp. 129–160, 1979. doi:10.1007/BF00330404
16. J. W. Cooley, J. W. Tukey, "An algorithm for the machine calculation of complex Fourier series," *Mathematics of Computation*, vol. 19, no. 90, pp. 297–301, 1965. doi:10.1090/S0025-5718-1965-0178586-1
17. E. Y. Lam, J. W. Goodman, "A mathematical analysis of the DCT coefficient distributions for images," *IEEE Transactions on Image Processing*, vol. 9, no. 10, pp. 1661–1666, 2000. doi:10.1109/83.869177
18. G. J. Sullivan, "Efficient scalar quantization of exponential and Laplacian random variables," *IEEE Transactions on Information Theory*, vol. 42, no. 5, pp. 1365–1374, 1996. doi:10.1109/18.532878
19. J. L. Mannos, D. J. Sakrison, "The effects of a visual fidelity criterion on the encoding of images," *IEEE Transactions on Information Theory*, vol. 20, no. 4, pp. 525–536, 1974. doi:10.1109/TIT.1974.1055250
20. I. H. Witten, R. M. Neal, J. G. Cleary, "Arithmetic coding for data compression," *Communications of the ACM*, vol. 30, no. 6, pp. 520–540, 1987. doi:10.1145/214762.214771
21. D. Marpe, H. Schwarz, T. Wiegand, "Context-based adaptive binary arithmetic coding in the H.264/AVC video compression standard," *IEEE Transactions on Circuits and Systems for Video Technology*, vol. 13, no. 7, pp. 620–636, 2003. doi:10.1109/TCSVT.2003.815173
22. J. Teuhola, "A compression method for clustered bit-vectors," *Information Processing Letters*, vol. 7, no. 6, pp. 308–311, 1978. doi:10.1016/0020-0190(78)90024-8 (Exp-Golomb codes)
23. S. W. Golomb, "Run-length encodings," *IEEE Transactions on Information Theory*, vol. 12, no. 3, pp. 399–401, 1966. doi:10.1109/TIT.1966.1053907
24. G. N. N. Martin, "Range encoding: an algorithm for removing redundancy from a digitised message," Video & Data Recording Conference, Southampton, UK, 1979.
25. I. Pavlov, *LZMA SDK*. https://www.7-zip.org/sdk.html (range coder: `LzmaEnc.c`, `LzmaDec.c`)
26. S. Josefsson, *RFC 4648: The Base16, Base32, and Base64 Data Encodings*, IETF, 2006. (Section 5, base64url)
27. The Unicode Consortium, *The Unicode Standard*, Chapter 3, "Conjoining Jamo Behavior," and *Unicode Standard Annex #15: Unicode Normalization Forms*. https://www.unicode.org/ (Hangul syllables U+AC00–U+D7A3, normalization)
28. A. Ortega, K. Ramchandran, "Rate-distortion methods for image and video compression," *IEEE Signal Processing Magazine*, vol. 15, no. 6, pp. 23–50, 1998. doi:10.1109/79.733495
29. Ecma International, *ECMA-262: ECMAScript Language Specification*. https://tc39.es/ecma262/ (`Math.cos` is implementation-approximated)
30. T. Berners-Lee, R. Fielding, L. Masinter, *RFC 3986: Uniform Resource Identifier (URI): Generic Syntax*, IETF, 2005. (query, fragment and percent-encoding)
31. R. Fielding, M. Nottingham, J. Reschke, *RFC 9110: HTTP Semantics*, IETF, 2022. (Section 15.5.15, 414 URI Too Long)
32. GitHub Docs, "Managing a custom domain for your GitHub Pages site." https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site
33. Picsum Photos. https://picsum.photos/ (Unsplash photos. Tuning set ids 1015, 1025, 1062, 1084, 177, 237, 433, 64; validation set ids 1003, 1035, 1074, 200, 287, 334. The bear in the example image is id 433 (Thomas Lefebvre, https://unsplash.com/photos/aRXPJnXQ9lU))
34. Z. Wang, A. C. Bovik, "Mean squared error: Love it or leave it? A new look at signal fidelity measures," *IEEE Signal Processing Magazine*, vol. 26, no. 1, pp. 98–117, 2009. doi:10.1109/MSP.2008.930649
35. Q. Huynh-Thu, M. Ghanbari, "Scope of validity of PSNR in image/video quality assessment," *Electronics Letters*, vol. 44, no. 13, pp. 800–801, 2008. doi:10.1049/el:20080522
36. D. Goldberg, "What every computer scientist should know about floating-point arithmetic," *ACM Computing Surveys*, vol. 23, no. 1, pp. 5–48, 1991. doi:10.1145/103162.103163

### Measurements

- The 400-character limit of DCInside comments, the lack of display for images and `data:` links, autolinking only with `https://`, and ASCII codes after `?` being included in links while Hangul codes and anything after `#` are not: checked in an actual comment box.
- The 414 boundary of g.yeon.at: on 2026-10-26, requests with queries of 1,000–60,000 bytes returned 200 at 6,000 bytes and 414 at 9,000 bytes.
- Quality and speed figures: measured with Node.js 22.
