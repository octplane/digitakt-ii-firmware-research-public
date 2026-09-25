# OS 1.16: container census, bootstrap domain, and the MAIN boot chain

## Scope and evidence boundary

This dossier concerns **Digitakt II OS 1.16**, one version later than the
1.15C target of the rest of this repository. Everything below is `STATIC-AUTH`
unless labelled otherwise: it rests on a locally obtained update, an
independently written decoder, and static analysis checked against authentic
bytes.

Nothing here was executed on hardware. No image, decoded binary, disassembly,
string dump or authentication material is published.

### Tools referenced

- **[elektron-firmware-tool](https://github.com/mischa85/elektron-firmware-tool)**
  (MIT, by [@mischa85](https://github.com/mischa85)) — an independent C
  implementation of the SysEx transport, the ELE3 container and the LZ77 +
  Elias-gamma codec, in both directions. It is used below for the re-encoding
  result, and it served as an independent cross-check of the container census:
  same section identities and sizes, and decompressed sections byte-identical
  to a separately written decoder. Its `PROVENANCE.md` records it as a
  clean-room implementation containing no vendor code.
- A separately written decoder and decompressor, developed independently for
  this work before that tool was consulted, used for every other claim here.

| | |
|---|---|
| Firmware | Digitakt II OS 1.16 |
| Distribution zip SHA-256 | `dab77ff3a22490a638e309ddb3f0c1ba2f53826f0682c4d8b69e6167f0241958` |
| SysEx transport | 1,880,864 bytes |
| Decoded ELE3 container | 1,484,064 bytes |

## Container census, and the 1.15C delta

1.16 carries **six** sections where the published 1.15C census records five.
Sizes are decompressed unless marked raw.

| Section | Role | Size | Runtime destination |
|---|---|---|---|
| 5 | build-timestamp string | 15 B (raw) | — |
| 2 | bootstrap | 30,302 B | see below |
| 3 | ColdFire MAIN | 3,275,616 B | `0x40000400` |
| 4 | ColdFire updater | 32,776 B (raw) | `0x80000400` |
| 7 | SHARC loader/application | 321,016 B | — |
| 8 | unidentified | 159,948 B | — |

Comparing with the 1.15C table already published here: MAIN grows from
3,177,312 to 3,275,616 bytes, the updater stays at 32,776, and SHARC
loader/application content moves from 320,780 to 321,016. Sections 2, 5 and 8
are named here because the 1.16 section table enumerates them; whether the
1.15C container also carries them under a different census is not asserted.

The section table itself is a count followed by 16-byte
`(id, offset, size, load_address)` records. Its offsets are **container-relative**,
i.e. eight bytes less than the corresponding offset in the decoded file, which
is the same eight-byte transport preamble that carries the declared length and
the content checksum. Section 5's record points at the build-timestamp string,
which is how the record layout was confirmed rather than assumed.

## Section 2 is a distinct bootstrap domain

`STATIC-AUTH.` Section 2 is ColdFire code linked at a base of `0x800003FC`,
established by histogramming `pointer - string_offset` pairs: that base scores
168 string-start matches and every competing candidate scores approximately
zero. The section-table load address for this section is a staging coordinate,
not this link base.

This domain is not the updater (section 4). It contains the **STARTUP MENU**
reachable by holding `[FUNC]` at power-on, whose entries correspond one-to-one
with the officially documented recovery path, including the `[TRIG 4]` OS
UPGRADE mode named in the release notes. It also contains the bootstrap
self-upgrade path and the key/encoder/LED test screens.

It additionally owns the **diagnostic boot screen**: a text-only screen
reporting UI, DRAM and USB status, a UI firmware revision, the OS version and
the serial number. The OS version is obtained by reading the firmware header
still resident in nonvolatile storage and trimming its space-padded version
field before formatting — the OS version string is not embedded in this
section.

Architectural observation: boot-time screens are **split across images**.
Anything drawn before MAIN takes over belongs to section 2 or section 4, so a
search confined to MAIN will not find it. This cost real time here and is
recorded so others can skip it.

`NEGATIVE-BOUNDED.` Within section 2 no full-screen bitmap frame was found: a
1 KiB-block image test across the whole section returned no 128x64 candidate.
The diagnostic screen is composed from a 6-pixel proportional font whose width
table is byte-identical to one carried in MAIN.

## ColdFire MAIN: reset to first task

`STATIC-AUTH.` The updater hands control to MAIN by **dereferencing** the load
address rather than jumping to it: the longword at `0x40000400` is the entry
point. The chain from there is:

| Stage | Address | Behaviour |
|---|---|---|
| Reset entry | `0x400004E8` | saves the updater's argument, sets SP, configures MCF54415 peripherals, sets `VBR`, enables cache via `CACR`/`ACR0` |
| C runtime | `0x40196CFE` | clears BSS, runs an init call, installs default vectors, enables interrupts, executes `trap #0`, then spins |
| TRAP #0 handler | `0x40000410` | saves registers into the current task control block, selects the next task, `rte` |
| First task | `0x400CC780` | creates a second task and then idles permanently |
| Application main | `0x400CC864` | walks a static-constructor array, runs subsystem init, then branches on one bit of the updater-supplied argument |

The architectural point is that **MAIN is a multitasking RTOS entered through a
software trap**, with the user-facing application running as an ordinary task
rather than from a `main()` loop. This complements, and does not overlap, the
existing account of reset-time *updater* selection, which remains opaque here
too: this dossier begins after MAIN already has control.

`INFERENCE.` The bit tested against the updater-supplied argument at
`0x400CC864` plausibly distinguishes boot modes, but no controlled pair
establishes its meaning.

## The boot animation selector: five variants, but a deterministic choice

`STATIC-AUTH.` The startup animation is chosen at runtime. A selector reads a
15-bit linear congruential generator and walks a cascade of four comparisons,
choosing one of **five** draw routines held in a table of
`(parameter, function, threshold, flags)` records.

The stock thresholds give these **nominal weights**:

| Variant | Nominal weight |
|---|---|
| common | 99.902% |
| rare A | 0.049% |
| rare B | 0.024% |
| rare C | rarer |
| rare D | rarest |

**These are weights, not observed frequencies.** `STATIC-AUTH`: the generator's
state word is referenced from exactly one site in the whole image, inside the
generator itself. There is no seeding function and no other writer, so the draw
sequence is identical on every boot and the selector resolves to the **same
variant every time**. The weights describe what the cascade would do given a
uniform draw; the draw is not uniform across boots, it is fixed.

`HARDWARE, owner-reported.` An instrument running a build with these thresholds
re-weighted toward uniformity showed a single variant across 4-5 restarts, and
a different one from stock. That is the predicted consequence of re-weighting a
deterministic draw: the bucket boundaries move, so the same fixed value lands
elsewhere. Under a genuinely uniform draw, five identical boots would be roughly
a 0.2% event.

Consequently the four low-weight variants are best described as **unreachable
in practice on a given unit**, rather than as rare events. Reweighting alone
does not make them appear; changing which variant is selected does.

`HARDWARE, owner-reported.` A third build confirms this by changing the
mechanism rather than the weights. Replacing the generator's constant additive
term with a read of a free-running 32-bit hardware counter, in the same six
bytes, makes the intro vary between restarts. Three builds each discriminate:

| Build | Predicted | Observed |
|---|---|---|
| stock | one fixed variant | matches |
| thresholds re-weighted | one fixed variant, different from stock | matches |
| thresholds + live counter mixed into the generator | varies per boot | matches |

Had the generator been seeded all along, the first two would have varied; had
the counter been inactive, the third would not have. This is therefore a
confirmed diagnosis rather than a consistent one, and it is the evidence for
the unseeded-generator claim above.

The selected variant draws a 32-pixel-wide logo bitmap at a fixed position on
the 128x64 display. Its constructor arguments and its rendered extent disagree
about the bitmap's height, so the exact dimensions are not asserted here. The
rarest variant is a distinct dither path with its own source artwork and its
own translation unit, identified by a retained source-path string. The four
rare variants are, on the evidence, **boot easter eggs**.

This matters methodologically: artwork reachable only through the rare branches
will look unfamiliar to an owner and can be mistaken for dead content, or for
belonging to an unrelated feature, unless the selector is read first.

`NEGATIVE-BOUNDED.` Within MAIN only three genuine 8-bit grayscale images
exist, verified by a correlation test in which vertical correlation peaks
sharply at the declared width while text and code score near zero over the same
binary.

## Panic screens are pre-rendered, not composed

`STATIC-AUTH.` The crash/panic screen text exists **only as pixels**. The
strings shown on it are absent from the image as text. A panic path cannot
depend on the font and text-layout subsystem still being intact, so
pre-rendering the whole message is the expected design, and its absence from
the string inventory is a positive signal rather than a gap.

`INFERENCE.` A separate exception reporter formats a product-code-bearing
message with a program-counter field, consistent with the same path.

Caution for anyone repeating this: the global bitmap registry is built by one
very long straight-line constructor run with no intervening returns, so
**adjacency of two bitmaps in that run carries no feature relationship**. Two
assets being neighbours there was briefly mistaken for evidence of shared
purpose during this work. The distinctions that do survive are structural —
whether a bitmap is constructed into an array or into a named global — and what
subsequently references it.

## Bitmap assets use two different bit layouts

`STATIC-AUTH.` Assets are not uniformly encoded. Full-screen frames resolve
only as column-major with reversed page order; other assets, including the
logo drawn by the common intro variant, resolve only as row-major, MSB-first.

An incorrect choice is not obviously wrong: it leaves 8-pixel-tall text
legible while corrupting anything taller, which is exactly the failure mode
that makes a wrong layout look plausible. Render both before concluding that an
asset is garbage.

## Container re-encoding and package construction

This section reports on two gates listed as `unproved` in
[modification-gates](modification-gates.md).

`STATIC-AUTH, offline only.` Using the independent MIT-licensed
[elektron-firmware-tool](https://github.com/mischa85/elektron-firmware-tool)
introduced above, a modified MAIN section was re-compressed and a complete container rebuilt,
with the transport preamble's content checksum and the ELE3 cryptographic
trailer recomputed. The rebuilt package self-verifies, and decompressing its
MAIN section returns the modified input byte-for-byte while the remaining
sections are unchanged.

That tool also independently reproduces the container census above: same
section identities, same sizes, and decompressed sections byte-identical to a
separately written decoder. It repacks an unmodified file byte-for-byte.

What this does and does not establish:

- It **does** establish that modified-section re-encoding and complete
  container construction are achievable offline, and that the integrity
  relations are recomputable rather than opaque.
- It does **not** establish installability, device acceptance, minimum-version
  or rollback policy, recovery, timing, audibility or safety. No hardware
  transaction was attempted. The gate dispositions for hardware execution,
  recovery and deployment are unchanged by this dossier.

No authentication secret, derived key or accepted modified update is published
here, and none is required to read this document.

## Earliest remaining opacity

- Whether the 1.15C container also enumerates sections 2, 5 and 8, which would
  make the census difference a reporting artefact rather than a version change.
- The meaning of the boot-mode bit tested at `0x400CC864`.
- Whether the generator is advanced a fixed number of times before the selector
  on every boot, or merely appears so. From a cleared BSS its first draw is `0`,
  which selects the common variant, so the observed non-common selection implies
  prior calls; there are eight call sites.
- Where the DMA timer whose 32-bit counter MAIN reads at `0xFC07000C` is
  enabled. The hardware result above shows it *is* running and free-running,
  since mixing it into the generator visibly randomises the boot intro. But no
  section writes its control registers, so it is configured either outside these
  three images or through an address this analysis could not resolve
  statically. Whether it runs is answered; where it is started is not.
- Section 8's role. It carries no ColdFire idioms and almost no image data.
- The bitmap constructor's exact signature. Its pushed dimension arguments do
  not agree with the rendered extent of at least one asset, so either the
  argument order or an implicit padding rule is still misread.
- What the four rare intro variants render, beyond which bitmap objects they
  reference.
- Everything downstream of offline package construction: acceptance, recovery
  and hardware behaviour remain untested and unclaimed.
