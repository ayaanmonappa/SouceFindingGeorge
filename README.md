# Find George: scaled pattern search in C and RISC-V

**Georgia Tech ECE 2035, Project 1** · Ayaan Monappa

A 64×64 pixel crowd contains exactly one face that matches a reference mugshot, George P. Burdell, possibly scaled up to 5×. Every other face is an impostor that differs in one or two features. The program must return George's bounding box.

I wrote it twice: first in **C** to design and test the algorithm, then by hand in **RISC-V assembly**, where it was graded against a baseline on three metrics: static code size, dynamic instruction count and storage.

<p>
  <img src="docs/mugshot.png" width="180" alt="George's 12x12 mugshot">
  &nbsp;&nbsp;
  <img src="docs/crowd-sample.png" width="260" alt="A sample crowd">
  &nbsp;&nbsp;
  <img src="docs/answer-boxes.png" width="260" alt="Crowd with George's bounding box marked in green">
</p>

*Left to right: the mugshot, a sample crowd, and the grader's view with George's box in green. Figures are from the ECE 2035 project spec.*

---

## Results (RISC-V version)

| Metric | Baseline | Mine | |
|---|---|---|---|
| Static code size | 90 instructions | **129** | larger, see trade-off below |
| Dynamic instructions (average per crowd) | 3,000 | **~1,555** | **48% fewer** |
| Storage (registers + memory words) | 25 | **~28** | 9 of them are the key-cell table |
| Correct on test crowds | | **100%** | thousands of generated crowds |

I deliberately spent static size to buy speed. Dynamic count is the largest grading category, and the techniques that cut it most (a sparse scan, reading only 9 cells per candidate, skipping past rejected faces) each cost a few instructions of code.

---

## The problem

- The crowd is 4,096 bytes in row-major order, one byte per pixel, using colour codes 1–11. Face colours are 1–8 and background is 9–11.
- An unscaled face fits in a 12×12 box. At scale *n* it is 12*n* × 12*n*, with *n* from 1 to 5.
- Faces never overlap and never hang off the edge, and each face's box ends in *n* columns of pure background.
- Impostors vary by colour (hat, shirt, skin, eyes, mouth, emblem) and by structure (stripes instead of a "T", glasses, a flat mouth instead of a smile).

<p>
  <img src="docs/crowd-scaled-2x.png" width="260" alt="Crowd with George at 2x">
  &nbsp;&nbsp;
  <img src="docs/crowd-scaled-3x.png" width="260" alt="Crowd with George at 3x">
</p>

*George at 2× and 3×.*

A brute-force approach slides a 12*n* template over every position for every scale, which is roughly 4,096 positions × 5 scales × up to 144 cell comparisons. My approach reads a few hundred pixels instead.

---

## Algorithm

The search runs in four stages. Each stage uses a geometric fact about the faces to avoid reading pixels.

### 1. Sparse scan: rows 0, 12, 24… and every 3rd column

Every face is at least 12 pixels tall, so any set of rows spaced 12 apart crosses every face at least once. Within a row, the narrowest horizontal slice of a face is the chin, 3 pixels wide at 1×, so stepping 3 columns can't skip over any face.

That covers the whole 64×64 image in about **6 rows × 22 columns ≈ 130 reads**.

Each read is tested against George's four colours (red 2, blue 3, yellow 5, light blue 6) with a **bitmask**: `(108 >> colour) & 1`, where 108 = `0b1101100` has bits 2, 3, 5 and 6 set. That is 3 instructions per pixel instead of the 8 needed for four compare-and-branch pairs, and this test runs on every scanned pixel, making it the hottest code in the program.

### 2. Walk to a known landmark

From whatever pixel the scan lands on, a "walker" slides left until it hits background. When it is blocked, it probes down and then up the current column for a row that reaches further left, and continues from there.

Every face has only two places the walk can end:

- the **left end of the hat brim** (mugshot row 4, column 0), or
- the **left end of the collar** (row 11, column 2), because the chin above the collar is narrower and leaves a gap.

### 3. One width measurement gives scale and position

At the stopping point, the walker counts the width *w* of its row:

- Brim: *w* = 11*n*, which can only be 11, 22, 33, 44 or 55.
- Collar: *w* = 7*n*, which can only be 7, 14, 21, 28 or 35.

The two sets never overlap, so **a single width measurement identifies both the landmark and the scale**. The code then climbs to the top of that band and computes the box:

| Landmark | Scale | Top row T | Left column L |
|---|---|---|---|
| Brim | *n* = *w* / 11 | top − 4*n* | *x* |
| Collar | *n* = *w* / 7 | top − 11*n* | *x* − 2*n* |

In assembly the search for *n* tries *n* = 1, 2, 3… and stops when 7*n* > *w*. It builds 11*n* and 12*n* from shifts and adds, so no multiply instruction is needed.

**Guarding against a wrong walk:** if another face sits directly above or below, the vertical probe can wander onto it. The computed box must therefore contain the pixel the scan originally landed on. If it doesn't, the walk retries looking up only, and if that also fails, the scan moves on.

### 4. Confirm with 9 key cells, not 144

Instead of comparing the whole 12×12 mugshot, the program checks the **9 cells that cover every possible variation**, ordered so the most common differences are tested first:

| Order | Mugshot cell (row, col) | Expected colour | Catches |
|---|---|---|---|
| 1 | (7, 3) smile corner | red | flat mouth (50% of faces) |
| 2 | (2, 5) stem of the "T" | yellow | striped emblem (40%) |
| 3 | (5, 5) between the eyes | skin yellow | glasses (45%) |
| 4 | (5, 3) beside left eye | skin yellow | glasses |
| 5 | (8, 5) middle of mouth | red | mouth colour |
| 6 | (5, 4) left eye | light blue | eye colour |
| 7 | (6, 5) cheek | yellow | skin colour |
| 8 | (0, 5) hat | blue | hat colour |
| 9 | (11, 5) shirt | blue | shirt colour |

Cell (*i*, *j*) of a face at scale *n* is at crowd offset (T + *in*)·64 + (L + *jn*). The check stops at the first mismatch, so **most impostors are rejected after one or two reads**.

In assembly, each table entry packs position and colour into one word, `(64·row + col)·16 + colour`, so a single load gives both. The offset is scaled by *n* from the face's top-left address.

### Avoiding repeated work

- **Skip rejected faces:** after a face fails, the scan column jumps to L + 12*n*, past the face's box.
- **No duplicate checks without storing state:** a face of height 12*n* can be crossed by several scan rows. It is only tested on the **first** scan row that crosses it (`r − 12 < T`), and skipped on later ones. This avoids keeping a list of already-checked faces in memory.
- **Early exit:** George appears exactly once, so the program stops at the first match and needs no "not found" path or end-of-image bound in the main loop.

---

## Assembly optimizations

After the first working RISC-V version, I optimized in measured steps. I re-tested against thousands of generated crowds after every change.

| Change | Effect |
|---|---|
| Merged the mirror-image look-down and look-up loops into one loop, with direction (+64 / −64) and stop row held in registers | about 14 fewer static instructions |
| Removed the four "box fits on image" tests, which the containment check already covers | 4 fewer static instructions |
| Removed the main loop's end-of-image bound, since George is guaranteed to exist | 1 fewer instruction per scanned row |
| Reordered blocks so common paths fall through instead of jumping | fewer taken jumps |
| Computed key-cell and answer addresses from the face's box address instead of recomputing row × 64 + column | fewer instructions per candidate |
| Rotated the hot scan loop and merged registers with non-overlapping lifetimes | about 300 fewer dynamic instructions per crowd, and one fewer register |

The final answer is packed as `TopLeft << 16 | BottomRight` and submitted with a single `ecall`.

---

## Source code

This is a graded course project, so the code lives in a **private repository** under Georgia Tech's academic integrity policy:

**[github.com/YOUR-USERNAME/YOUR-PRIVATE-REPO](https://[github.com/ayaanmonappa/DynamicFindingGeorge](https://github.com/ayaanmonappa/DynamicFindingGeorge))** *(private, available on request)*

It contains:

| File | Contents |
|---|---|
| `P1-1.c` | C implementation (design and correctness reference) |
| `P1-2.asm` | RISC-V implementation, with a dated change log of every optimization |

**To request access (to employers interested in the code)**, email [ayaanmonappa20@gmail.com](mailto:ayaanmonappa20@gmail.com?subject=Access%20to%20Find%20George%20source) with your GitHub username and I'll add you as a collaborator.

## Running it

**C version:**

```bash
gcc -Wall -o P1-1 P1-1.c
./P1-1 test_4_1037_2532.txt      # prints George's top-left and bottom-right pixel offsets
```

Each test file is a crowd dump, and its name contains the expected answer.

**RISC-V version:** runs in ERNIE, the course's RISC-V emulator. ERNIE generates a random crowd, runs the program and draws the returned box over the image (green if correct).
