# FS-CUDA — changelog

User-facing notes, newest first. The in-app updater fetches this file from the
public repo and shows every section newer than the version you are running, so
each heading must start with `## <version>` and sections must stay in
descending version order.

## 1.11.4

**Most stacks are now noticeably faster, and nothing about the results changes.**

- **Stacks between 16 and about 130 frames were using the slower engine.** They
  no longer do. FS-CUDA picks between two engines by frame count, and the
  threshold was set back when the fast one genuinely lost on small stacks. It
  has not lost for a long time — it has since taken on monochrome, normalization,
  autocrop, resume, per-pixel weighting and trail rejection, and got quicker at
  all of them — but the threshold was never revisited.

  Re-measured on real frames, both engines, identical masters throughout:

  | frames | before | now |
  |---|---|---|
  | 32 | 7.9 s | **4.9 s** |
  | 64 | 10.0 s | **9.0 s** |
  | 128 | 25.5 s | **16.8 s** |
  | 200 | 41.2 s | **26.1 s** |

  Below 16 frames the old engine really is quicker, and still gets the work.

- **Large stacks are a little faster too**, from removing redundant work in the
  trail pre-pass: 1089 frames with the full quality set went 3 min 55 s to
  3 min 39 s.

- A fix worth naming even though you would never have seen it: when two
  candidate trails tied on exactly equal evidence, which one got masked depended
  on an ordering the code never guaranteed. Same frames in, same measurements,
  potentially different pixels out. It is now decided explicitly.

Your masters are unchanged — byte for byte, on 1089 real frames, on every path.

## 1.11.3

**Large stacks on the CPU path could die without saying anything. Fixed.**

On a big run — 1089 frames with Normalize, Autocrop, RGB align and Per-pixel
weight — the program could vanish partway through registering, with no error
message and nothing saved. It needed a machine with plenty of free memory to
show up, which is why it survived this long.

Three separate faults, all now fixed:

- **It was claiming too much memory.** The working budget was set from the free
  memory reported at startup, as though the whole of it were available to the
  program. It is not: while stacking, Windows is also caching the frames being
  read and the working file being written. On a 64 GB machine that meant asking
  for about 52 GB and eventually being refused one allocation too many. It now
  leaves the cache room to work — and the run is *faster* that way, not slower:
  6 min 38 s against 7 min 35 s when forced smaller by hand.

- **When memory did run short, it gave up instead of using the disk.** Frames
  are held in RAM up to a limit and spilled to a working file beyond it. If the
  RAM side failed, the run ended — with the working file sitting there unused.
  It now simply stops growing and spills from that point on.

- **The error was being destroyed on its way out.** A background thread was
  being torn down as the failure propagated, which on Windows ends the process
  immediately and discards the message. That is why there was nothing to report.
  Any failure now reaches you as an error instead of a disappearance.

If you have hit an unexplained exit on a large stack, this was very likely it.
Nothing about image results changes.

## 1.11.2

**The "Slower engine (v1)" warning was still blaming Reject trails.**

If you ticked Reject trails, the options panel told you the run had dropped onto
the slower engine. It had not — 1.11.0 moved trail rejection onto the fast one
and the warning was never updated. So for two releases the program argued
against an option that now costs about 30 seconds on a 1089-frame stack instead
of roughly doubling it.

Nothing about stacking changes. If you turned Reject trails off because of that
message, you can turn it back on.

The warning and the engine now read from the same place, so it cannot say one
thing while the program does another. That had happened twice: SuperPixel was
missing from the list entirely, and then trail rejection stayed on it after
moving. A test now checks the claim against the real engine rather than against
a second copy of the rule.

## 1.11.1

Two options that were removed from the window some releases ago were still
described in the manual as though you could pick them, and were still reachable
from the command line. Both are now gone entirely, along with every mention of
them.

- **The second debayer method** — retired in 1.5.13 because RCD beat it on every
  measure, but the manual still listed it beside RCD and SuperPixel.
- **The sharper resampling option** — retired for the same reason. Measured on
  900 real frames it sharpened nothing (FWHM 9.34 against the default's 9.31)
  while costing stars and adding noise.

Your masters are unchanged: both were off by default and the code that remains
produces bit-identical output — every engine-parity check still reads exactly
zero. Existing settings files still load.

## 1.11.0

**Trail rejection is about twice as fast, and produces exactly the same master.**

Nothing about the trails themselves has changed — the same frames are flagged,
the same pixels masked, and the resulting file is byte-for-byte identical to the
one 1.10.0 produced. This release is purely about what ticking the box cost you.

- **"Reject trails" no longer drops the whole run onto the slower engine.**
  FS-CUDA has two stacking engines, and the fast one could not do trail
  rejection, so switching the option on quietly switched engines too. Measured
  on 300 real frames: **75.7 s → 35.3 s**. With local normalization as well,
  **93.7 s → 50.3 s**.

  The fast engine never builds a whole registered frame in memory — that is why
  it is fast, and why it could not run a detector that needs one. It now warps
  each frame through the same GPU path it already uses for stacking, hands that
  to the unchanged detector, and keeps only the list of pixels to mask.

- **Local normalization + trail rejection now works on the fast engine too.**
  This combination was the last one still forced onto the slow path.

- Three further speedups inside the new detection pass, worth 20.9 s → 9.2 s of
  it on those 300 frames: recycled frame buffers instead of allocating and
  clearing 32 GB of memory that was immediately overwritten, page-locked
  transfers, and copying back only the brightness the detector actually reads
  rather than all three colour channels.

If you do not use trail rejection, this release changes nothing for you.

## 1.10.0

Trail rejection was rebuilt. On the test data it went from finding **none** of
the satellite trails you could see in the frames to finding **all three**, while
throwing away far less of your night.

- **Trails now remove PIXELS, not whole frames.** A trail is about 0.1% of a
  frame; the old behaviour discarded the other 99.9% with it. Measured on 38
  real subs: **18,608 pixels masked instead of three whole frames — 2,642× less
  data lost**, the same trails gone, and **11 stars recovered** in the master.
  Quality now equals not rejecting at all.

  The masked pixels are simply absent for that one frame, so the others supply
  them. Nothing is invented — this is not the old "patch it with the reference",
  which is what once carved a band through M57.

- **Faint trails are detectable at all now.** The line detector's angle grid was
  1.5°, and across a 4,650 px frame a half-cell error moves a line by 61 px — so
  a faint trail's evidence smeared across 30 cells and could never register.
  Worked through: 765 votes total, 25 in the best cell, against a threshold of
  82. It was mathematically impossible, which is why raising sensitivity had
  never helped. The grid is now four times finer for the same memory.

- **Diffraction spikes are no longer mistaken for trails.** Brightness cannot
  tell them apart — on real frames the spike from a bright star just outside the
  field was the *strongest* line in most subs, brighter than a genuine satellite
  elsewhere. A spike radiates *from a star*, so lines passing through one are
  now spared, at any angle.

- **The width test follows your seeing.** A trail is blurred by the same optics
  as the stars, so it can never be thinner than they are. A fixed 6-pixel limit
  sat below a 7.04 px FWHM and was rejecting real trails for being exactly as
  wide as they must be — one of them by a single pixel.

- **Same detector on every path.** Trail behaviour used to change depending on
  whether Drizzle was ticked: two unrelated options, one outcome. Fixed.

- **You can see what it did.** The result panel now reports how many frames
  carried a trail and how many pixels were masked. Nothing is dropped, so
  "Rejected 0" was previously all you saw.

- **New: Linear fit rejection.** Clips each pixel against a line fitted to its
  sorted samples rather than against the median. Reach for it when a session
  drifted in transparency or sky level, where sigma clipping's threshold widens
  and lets outliers hide. On a stack with no drift it is *worse* — measured 924
  stars and noise 29.0 against 938 and 27.3 — so it stays off by default and the
  tooltip says so.

## 1.9.1

- **The narrowband colour products are about twice as fast.** Asking for SHO,
  HaRGB and RGB_SHO together cost **8.1 s; it now costs 4.0 s** — measured on the
  same machine against the 1.9.0 build, not estimated.

  Each product used to register the narrowband masters itself, onto the same
  pixel grid as the others — eight registrations where three do, plus the same
  16 megapixel star detection repeated eight times. They are registered once now
  and shared.

  Three smaller ones with them: finding a channel's sky level no longer
  duplicates the whole 65 MB image to read one number out of it, the SHO channel
  balance takes two order statistics instead of sorting 3.2 million samples three
  times, and the colour smoothing now uses every core like the rest of the
  pipeline.

- Your files are unchanged in all but the last decimal. The single-filter masters
  are byte-identical; the colour products move by at most 0.3 ADU on an 840 ADU
  sky, from the sampled median — 2% of one step of the 16 ADU your camera
  quantises to.

- **New in the manual: how to tell whether to bin or to drizzle.** Work out your
  plate scale and pixels-per-FWHM, and the answer follows. Above ~4 px per FWHM
  you are oversampled and **Output binning 2×2 is a free doubling of
  signal-to-noise**; below ~2 you are undersampled and drizzle genuinely helps.
  Measured on a real 1325 mm / 3.8 µm rig at 5.7 px per FWHM, drizzling would
  have made the result *worse* — more pixels, no more detail, noise spread
  thinner. Both languages, with the arithmetic.

## 1.9.0

- **New: `<name>_RGB_SHO.fits`** — broadband **colour** with narrowband
  **structure**. The one to reach for when the SHO shows detail your RGB does
  not. A luminance is built from all six filters, weighted by how well each
  measured, and replaces the colour master's brightness; colour and star colour
  stay broadband.

  Measured on real M1 masters it comes out better than the plain colour master on
  every channel: sky noise 12.9 → 11.8 and nebula SNR 27.3 → 29.0 in red, with
  green and blue similar — and it carries the filament structure the broadband
  continuum was hiding.

  Expect a measured improvement, not a dramatic one. SHO looks striking because
  it discards the continuum entirely, which is also what makes it unnatural;
  RGB_SHO keeps the true rendition.

- Two things this needed, both found by measuring rather than assuming:

  Building the luminance from **narrowband alone wrecks the image**. Replacing a
  luminance replaces the brightness outright, so the Crab took the narrowband's
  amplitude — 60 ADU above sky against broadband's 360 — and nebula SNR fell from
  27 to 3.7. The structure survived and the contrast did not. All six filters
  together fix it.

  The **colour is smoothed** slightly before the swap while the luminance stays
  sharp. Scaling by a luminance injects its noise into all three channels, which
  showed as coloured speckle across the background; colour carries no fine detail
  the eye can resolve, the same reason image codecs subsample it.

- **The SHO channels are now balanced.** Sky backgrounds were already matched to
  a common zero point; each channel is now also linearly rescaled so its signal
  spans the same range as the others. Without it a SHO is dominated by whichever
  line is strongest — Ha carried 3.5× the signal of SII and OIII on real data,
  which is exactly why an unbalanced SHO comes out uniformly green.

  The rescale is `out = a × in + b` — strictly linear, so nothing is clipped or
  gamma-shifted and downstream work is unaffected. The gains are printed in the
  run log. This is the standard mechanical step, not a colour calibration.

- Command line: `--rgb-sho`, implying `--combine`.

## 1.8.0

- **New: a colour SHO master.** The Hubble palette — **SII → red, Ha → green,
  OIII → blue** — written as `<name>_SHO.fits`. Without SII it falls back to
  **HOO** (Ha → red, OIII → green and blue) and is named `_HOO`, so which mapping
  you got is never in doubt. Without OIII neither exists and nothing is written.

  Expect a strong green cast from raw SHO. That is correct: on a typical emission
  target Ha is several times stronger than the other two, so green dominates. The
  sky backgrounds are matched for you; balancing channel *gain* is a colour
  decision and stays yours — a linear fit then SCNR is the usual next step.

- **New: RGB deepened with narrowband,** `<name>_HaRGB.fits`. Ha is added to red
  and OIII to blue, green untouched, so you keep the real colour and the star
  colour only broadband can give while narrowband carries the nebula.

  The blend is a background-matched maximum, not a sum. A star appears in both
  filters and adding them inflates it (10000 + 3000); the maximum leaves it at
  whatever the better filter measured, and nebulosity — far brighter in
  narrowband — wins on its own with no ratio to invent.

- **All three colour outputs share one pixel grid.** The colour master, SHO and
  HaRGB overlay exactly as layers, and a single plate solution describes all
  three. The single-filter masters keep their own grids and carry no WCS, because
  each is stacked against its own reference.

- **A meridian flip between sessions is handled.** Narrowband shot on the other
  side of the meridian, 180° from your broadband, registers correctly — matching
  is on star patterns and rotation-invariant. Verified on real frames: rotating
  half a set by 180° reproduced the unrotated master exactly.

- A palette is a convention, not a measurement, so neither output is ever
  produced silently: each is named for exactly what it is, and the run reports
  the mapping it used.

- Command line: `--sho` and `--hargb`, both implying `--combine`.

## 1.7.0

- **An LRGB batch no longer writes anything until you press Save.** It used to
  write every master the moment each filter finished — up to seven files per run,
  before you had seen a single one of them. They are now held in memory and
  written only when you export.

  This matters beyond tidiness. FS-CUDA is donationware, and both the export
  watermark and the trial counter live in the Save handler. Files written by the
  batch went around both, so a batch produced unwatermarked, uncounted exports.

- **Red, green and blue are always combined into one colour master.** The
  *Combine to RGB* checkbox is gone: the combine needs all three and already
  tells you when they are missing, so there was never a decision to make.

- **Save writes one colour master, not a redundant pile.** You get `<name>.fits`
  — plate-solved when Plate solve is on — plus only the filters the colour master
  cannot contain: `_L`, `_Ha`, `_OIII`, `_SII`. Luminance is only the
  registration reference and narrowband never takes part, so those would
  otherwise be lost, and a narrowband-only night still exports everything.

  There is no `_R`, `_G` or `_B` file any more. That data is already inside the
  colour master; the old `_RGB.fits` was a byte-identical duplicate of what Save
  wrote next to it.

- **One press is one export.** However many files a Save writes, it counts once
  against the trial — an LRGB user was otherwise burning it several times faster
  than an OSC user for the same single night.

- Astrometry is attached to the colour master only. Each filter is stacked
  against its own reference, so the narrowband masters do not share the colour
  master's pixel grid; copying its solution onto them would put every star in the
  wrong place while looking perfectly valid.

- The command line is unchanged: `fs_stack --lrgb` has no Save button, so it
  still writes every master itself.

## 1.6.9

- **Fixed: "Darks Light" was clipped.** Same fault as "Lights OIII" in 1.6.8, in
  a second place the fix had missed — a hard-coded column position. Every label
  in the folder panel now measures itself.

- **Fixed: an LRGB batch with Frame selection *Off* ignored your settings.**
  Reject trails, Normalize, Autocrop, per-pixel weighting and the rejection and
  weighting modes were all silently dropped and the stack ran with defaults.
  Turning frame selection on happened to hide the bug. Only LRGB batches were
  affected.

- **Improved: trail rejection measures candidate trails correctly.** The
  detector locates candidates on a coarse 1.5° grid, then measured each one at
  that grid position. Across a 4650 px frame a half-cell angle error puts the
  line 61 px away from where it looks, so it was measuring blank sky: a real
  satellite track came out 4–5× too faint and 2–3× too wide, failing both tests
  it should have passed. Each candidate is now fitted to its own pixels first,
  and width is measured across the track instead of counting every noise wiggle
  in the frame.

  This changes what the detector *sees*, not yet what it *drops* — flagged sets
  are unchanged on 239 real frames across two targets. It is the groundwork for
  catching faint trails, which needs a further change.

  Note what trail rejection cannot do: it drops frames that disagree with the
  others, so it only removes a trail present in a *few* frames. A diffraction
  spike or reflection from a bright star lands in the same place in every frame
  and no amount of frame rejection will remove it — that one is fixed at the
  telescope, not in software.

## 1.6.8

- **New: a dark per narrowband filter.** The single **Darks** row is now **Darks
  Light**, serving L, R, G and B, and **Darks Ha**, **Darks OIII** and **Darks
  SII** sit beside it.

  This is not tidiness. A dark only cancels the sensor's own signal for the
  *same* exposure, and narrowband subs are routinely 300 s or 600 s where
  broadband is 120 s. Calibrating a 300 s Ha sub with a 120 s dark leaves most of
  the dark current behind. Leave a narrowband Darks row empty and that filter
  falls back to Darks Light, so nothing changes if you shoot everything at one
  exposure.

- **Fixed: "Lights OIII" was clipped to "Lights OII".** The label column was a
  fixed width from before narrowband existed. It is now measured from the widest
  label actually drawn, so it survives longer names, a font change or a different
  display scale.

## 1.6.7

- **New: Ha, OIII and SII stack in the LRGB batch.** Narrowband filters were
  skipped with a warning; they now have their own slots and produce `<name>_Ha`,
  `<name>_OIII` and `<name>_SII` alongside the broadband masters. The usual
  spellings are all recognised — `Ha`, `H-alpha`, `H_Alpha`, `OIII`, `O3`, `SII`,
  `S2` — in the FITS `FILTER` header or as their own folders.

- **Restored: Lights L and Flat L.** The per-filter layout now has a row for
  every filter: L, R, G, B, Ha, OIII, SII. Leave a row empty for a filter you did
  not shoot.

- **Narrowband stacks but does not become a colour image.** Combine to RGB still
  needs red, green and blue. SHO and HOO are conventions rather than facts, and
  there is no single correct mapping from three narrowband filters to three
  colour channels — choosing one inside FS-CUDA would bury a decision you could
  not see or change. You get the masters; the palette belongs to your processing
  software.

  A filter this version does not recognise is still reported rather than guessed,
  so an unlabelled wheel slot never lands in a channel by accident.

## 1.6.6

- **Fixed: Frame selection now works for LRGB batches.** The setting was visible
  and changeable during an LRGB run and did nothing at all — a control that
  ignores you is worse than one that is not there. Review now happens for a batch
  exactly as it does for a single stack: measure, choose, then stack.

  Every filter's frames appear in **one** review window, tagged `[R]`, `[G]`,
  `[B]`, so you judge the whole night at once instead of one dialog per filter.
  Your choices are handed back to each filter separately.

- **New: the kept count is broken down per filter.** This matters more than it
  sounds. The filters do not share a distribution — measured on a real session,
  frame FWHM ran 6.0–7.1 in red, 7.6–7.7 in green and 7.8–7.9 in blue. One
  threshold applies to all of them, so `FWHM < 7.5` keeps most of red, cuts most
  of green and removes **every blue frame**, while the overall "kept" number
  still looks healthy and Combine to RGB then has no blue channel to work with.

  The review now shows `R 190/201  G 195/202  B 4/194`, turns red when a filter
  falls below three frames, and says outright that blue is usually the softest
  and noisiest. Set thresholds loose enough for your worst filter, or judge from
  the histograms.

  With Frame selection **Off**, nothing changed and nothing extra is read.

## 1.6.5

- **New: a Manual button in the header.** The full user manual — every option
  explained, both languages — now opens inside FS-CUDA instead of living only on
  the web page. It is built into the program, so it works with no internet
  connection and always matches the version you are running. There is a search
  box, because it is a long document.

  The manual now also covers monochrome cameras, the LRGB batch and Combine to
  RGB, and explains what the plate solve checks before it will write coordinates.

## 1.6.4

- **Fixed: no more console window flashing during Save FITS.** The plate solve
  and the target search fetched their data through a command shell, which opened
  a black window on screen for the length of the call. They now run without one.

- **Fixed: the plate solve could write a wrong sky position and call it a
  success.** If the Focal or Pixel size was wrong, the search ran at the wrong
  plate scale and could still settle on a handful of stars — reporting an
  excellent RMS, because a fit with six stars has no room left to be wrong in.
  That solution went into the file and only surfaced later, in another program:
  a real case wrote a position at half the true scale from six stars, and
  a spectrophotometric colour calibration then reported "only 0 samples are available"
  because every catalogue star landed on empty sky.

  A solution now has to rest on at least 15 matched stars. Below that it is
  refused and **no coordinates are written at all** — a wrong position is worse
  than none, because everything downstream believes it. A solve that only just
  clears the bar is accepted but says so.

- **Fixed: Focal and Pixel size are filled in from the image.** They are in the
  FITS header your capture software already writes (`FOCALLEN`, `XPIXSZ`), so
  they are read from the finished master instead of being typed in and left
  stale. This is what caused the case above: 2800 mm still in the box for a
  1325 mm scope.

- **Improved: when a solve fails it now says "Please check Focal and Pixel
  size".** Wrong optics are by far the most common cause, and the old message
  named no suspect.

- **Faster: monochrome stacking again, about 1.3x on top of 1.6.3.** Frames are
  now calibrated in parallel rather than one at a time. A 201-frame filter went
  from 25.0 s to 19.7 s — twice as fast as before 1.6.3 — and the master is
  still byte-identical.

## 1.6.3

- **Monochrome stacking is now GPU-fused, and about 1.6x faster.** Mono used to
  fall back to the classic engine. It no longer does: the GPU band kernels carry
  a single channel and skip the demosaic entirely, which also lets each band be
  roughly three times taller for the same video memory.

  Measured on a real 597-frame LRGB session (4656 × 3520): **2 min 6 s → 1 min
  16 s** for three masters plus the combined RGB. A single 201-frame filter went
  from 39.7 s to 25.0 s.

  **The masters are byte-identical to the ones the old path produced.** This is
  a speed change and nothing else — same rejection, same weighting, same pixels,
  verified by comparing whole files, not just statistics.

  Colour stacking is untouched.

- **Corrected: the engine warning claimed the GPU was idle when it was not.**
  The Options panel said "CPU path" for six different settings, and the tooltip
  went further with "the GPU stays mostly idle". The classic engine still
  integrates on the GPU, and for colour data calibrates and debayers there too —
  what it gives up is keeping every stage in video memory. It now reads "Slower
  engine (v1)" and explains which stages run where. Mono has been removed from
  that list, because it is no longer one of them.

## 1.6.2

- **Simpler per-filter layout: R, G and B only.** The *A folder per filter*
  layout no longer shows Lights L and Flat L, so it is seven rows instead of
  nine. If you had set an L folder in 1.6.1 it is cleared, so it cannot keep
  feeding the run from a row you can no longer see.

  Luminance itself still works: *One folder, split by FITS FILTER header* finds
  an L filter on its own and applies it as luminance when combining.

## 1.6.1

- **New: choose how your filters are organised.** LRGB batch now offers both
  layouts, picked with a radio button:

  *A folder per filter* — give each filter its own lights folder and its own
  flat, with one shared Darks row. A filter you did not shoot: leave its row
  empty.

  *One folder, split by FITS FILTER header* — for a session captured straight
  into a single directory, where each frame's FILTER card decides its channel.
  This is what 1.6.0 did, and it stays the right choice when hand-sorting
  hundreds of frames is the alternative.

- **Fixed: a Flat given as a FOLDER failed the whole run.** The per-filter Flat
  rows are folder pickers and the setting was always documented as accepting a
  folder or a single master, but a folder was opened as if it were one file —
  so every filter died with "FITS: cannot open". A folder of raw flats is now
  integrated into a master, exactly as the single-stack path has always done.
  The same fix applies to Darks.

## 1.6.0

- **New: monochrome camera support.** Tick *Mono / LRGB frames* in Input and
  nothing is demosaiced — the master stays a single channel at full sensor
  resolution. Bayer drizzle and RGB align switch themselves off (they mean
  nothing without a colour filter array) and cosmetic repair uses the adjacent
  eight pixels instead of the same-colour neighbours two pixels away. Note that
  mono runs on the v1 engine: the fused band kernels demosaic in hardware and
  have no mono path, so expect v1 timings.

- **New: LRGB batch — every filter in one press.** Tick *LRGB batch* and Lights
  becomes ONE folder holding all your filters. Each frame's FITS `FILTER` card
  decides its channel, so nothing needs hand-sorting; the folder is searched
  recursively, so per-filter subfolders work just as well. Darks is shared
  (darks do not depend on the filter) and Flats is a folder of master flats,
  matched to each filter the same way. One press writes `<name>_L`, `_R`, `_G`,
  `_B`. Measured on a real 597-frame session: 2.1 minutes, all 597 frames used.

  Frames marked dark, flat or bias by `IMAGETYP` are skipped, so pointing
  Lights at the session root does not stack your calibration frames. Anything
  with no recognisable filter is **reported, never guessed** — treating Ha as
  red would produce an image that looked plausible and was wrong. Press *Scan
  filters* to see the split before committing to a long run.

  A filter that fails does not discard the others: losing three hours of R, G
  and B because L had one unreadable frame is the wrong behaviour.

- **New: Combine to RGB.** After stacking each filter, the masters are
  registered onto a common frame and written as one colour image `<name>_RGB`.
  L is applied as luminance when present, otherwise G is the reference (it has
  the most stars, which gives the weakest channel its best chance). Channel
  backgrounds are matched by **offset**, which removes the filter-to-filter
  pedestal without inventing a colour balance. Edges where the channels do not
  overlap are trimmed, and the per-channel FWHM is reported — if the channels
  differ in sharpness by more than 30% you are told, because combining
  unmatched star profiles puts coloured halos on bright stars.

  **The result is linear and registered but NOT colour-calibrated.** Making red
  mean red needs photometry against a star catalogue; do the colour calibration
  in your processing software.

- **Fixed: a light list written by a Windows editor lost its first frame.**
  Notepad, VS Code and PowerShell all write a byte-order mark at the head of a
  text file, and those three invisible bytes attached themselves to the first
  path — so one frame per filter could not be opened and the whole filter
  failed, reporting a corrupt file when the real problem was the text encoding.

- **Fixed: Autocrop and Combine to RGB could not be used together.** Autocrop
  trims each filter by its own dither, so the masters came out at slightly
  different sizes and the combine refused them. Registration never needed
  matching sizes — it matches star patterns, not pixels — so it now accepts
  them and trims to the overlap as it already did.

## 1.5.17

- **Fixed: Frame Selection left most frames unmeasured.** The analysis pass
  measured the first 8 frames, then skipped ahead and carried on from frame 64,
  so everything in between showed FWHM 0.000, Ecc 0.000 and 0 stars. Worse, a
  zero passes every "less than" filter, so those frames were silently KEPT by a
  rule like "FWHM < 7.5" and the counter reported far more frames kept than had
  actually been checked. On a 100-frame set, 56 frames were never measured.
  Stacking itself was never affected — it measures every frame through a
  different path — so your masters were fine; only the Frame Selection table
  and the choice it made were wrong.

## 1.5.16

- Frame Selection: the "PSF Sig Weight" filter label was cut off mid-word. The
  label column is now measured from the text rather than fixed, so it fits at
  any font size or display scaling.

## 1.5.15

- A shorter header. The name, version and licence line now sit in two compact
  rows beside the badge instead of four stacked ones, giving the controls
  column back about 28 pixels.

## 1.5.14

- **The default setup is written down.** The Options section names it, every
  tooltip says whether the default has that option on or off, and a Restore
  button puts everything back with one click. The default keeps the stack on the
  fused GPU engine, which is the fast path.

## 1.5.13

- **Every option now explains itself.** Hover any setting and you get what it
  does, what it costs, and when to use it — with the measured number wherever
  there is one. Each also says plainly whether it drops the stack off the fast
  GPU engine, which is the biggest speed lever in the program: 152 s against
  383 s on a 1089-frame run.
- The warning that names the options costing you the GPU engine was missing
  SuperPixel, so a stack running slowly because of the debayer blamed only the
  other settings.
- **Two options removed.** Both were choices that cannot win. RCD beats the
  older debayer on every count, and the sharper resampling option — offered as "sharper"
  — measured no sharpening at all (FWHM 9.34 against the default's 9.31) while
  costing stars (531 against 554) and noise (1.91 against 1.79). Existing
  settings files still load; a stored value for either resolves to the default.
- The autocrop coverage slider sat under Defect fix, two rows below the box it
  belongs to. It is now directly under Autocrop.
- **The preview stretch is fixed at its gentlest setting** and the Shadows and
  Midtones sliders are gone. Shadows set the black point below the sky, so a
  higher value clips less; at the maximum nothing real is driven to black, and
  the background lands in the same place either way because the midtones are
  solved to put it there. Compared both on a real 1089-frame master: same
  background, cleaner sky. Display only — it never changed your saved data.

## 1.5.12

- **Fixed: plate solving failed on SuperPixel masters** ("match failed"). The
  solver works out the expected image scale from your focal length and pixel
  size, then searches within a few percent of it — but it only knew that drizzle
  changes the master's pixel size, not that SuperPixel doubles it. So it hunted
  at half the true scale and could never match. Output binning had the same
  problem. The master now reports its own scale and the solver uses it, so any
  future setting that changes the geometry cannot break this again.

## 1.5.11

- **Fixed: SuperPixel together with Bayer drizzle produced a broken master** —
  a green, colour-noisy image, silently, with no warning. The two settings are
  alternatives, not a combination: SuperPixel avoids interpolation by combining
  each 2x2 sensor cell, Bayer drizzle avoids it by dropping the raw sensor
  samples. Asking for both left the drizzle canvas at half the size it should
  have been, so the colour planes were starved. Bayer drizzle now ignores the
  Debayer setting (nothing is debayered in that mode anyway) and the interface
  says so. If you want SuperPixel, untick Bayer.

## 1.5.10

- **Output binning.** A new setting combines each NxN block of the finished
  master into one pixel. If your pixel scale is finer than the seeing supports —
  the normal case for a long focal length on small pixels — the extra pixels are
  only spreading the same photons thinner, and binning trades that back for
  depth. Measured on 900 frames at 0.28"/px with 2.62" stars (9 pixels across a
  star, where 2 is enough): 3x3 binning took a faint galaxy from 0.77 to 1.44
  signal-to-noise per pixel, with stars still 3 pixels wide. Applied last, so it
  changes nothing about how the stack is built.
- Uncovered pixels are excluded from the binning average, so the crop border
  does not darken.

## 1.5.9

- **The Bayer pattern is read from your camera's header.** The interface never
  had a CFA setting, so every stack was debayered as RGGB. If your sensor is
  BGGR, GRBG or GBRG, your colours were wrong and there was no way to fix it.
  There is now a "CFA pattern" setting, defaulting to Auto, which reads
  BAYERPAT from the first light and tells you what it found.
- **Folders and file names with accents work.** An account name like
  `C:\Users\José\...` previously broke reading lights, writing the scratch file
  and saving the master.
- Files that cannot be read now say why. A tile-compressed (`.fz`) or
  multi-extension FITS is named as such instead of reporting a confusing
  dimension error, and a malformed header card names the keyword. `.fts` files
  are recognised alongside `.fits` and `.fit`.
- Trail rejection no longer costs a second pass over the whole stack. Frames are
  checked for satellite and plane trails while they are being registered, so a
  trailed frame is dropped before the first integration instead of after it.
  Around 250 s off a 900-frame run with trail rejection on. Measured over 300
  frames the master is unchanged by it: 512 → 511 stars, FWHM 8.91 → 8.93,
  noise 2.98 → 2.98.
- Large stacks use more of the graphics card. The GPU band was sized to 55% of
  free video memory regardless of stack size, which left a 900-frame run below
  the point where the card runs at full speed; it now takes up to 75% when — and
  only when — the band would otherwise be too short. 900 frames: 136.6 s →
  127.2 s, identical master. Smaller stacks are unaffected.
- The update check now runs by itself when the app starts and shows what changed
  before asking whether to install. The "Check for updates" button is gone.

## 1.5.8

- Autocrop was throwing away half the sensor. A pixel had to be covered by 90%
  of the stack to survive, which is nearly free on a perfectly tracked session
  and brutal on any session where the field drifts. The default is now 75%: on
  900 real frames the master went from 3.80 to 7.87 megapixels and the star
  count from 245 to 542, for 0.01 more noise and no change in FWHM. The slider
  added in 1.5.7 still lets you pick your own trade.

## 1.5.7

- The seeing exponent used by PSF weighting is chosen per stack instead of being
  fixed. Fixed, it let a handful of frames dominate and capped the effective
  depth at 256 of 900 frames on a session where the seeing improved: 42% less
  noise at 900 frames.
- Autocrop coverage threshold is exposed as a slider.

## 1.5.6

- Trail detection tests that a candidate is a *sharp bright* line. It was
  flagging 214 of 300 frames and carving a band through the target; now it flags
  10 of 300 and leaves the target alone.
- A frame carrying a trail is dropped from the stack rather than having its
  pixels overwritten with the reference, so trail repair can no longer leave
  scars.
- Plate solving works on drizzled masters again.

## 1.5.5

- Frames were being averaged in as zeros wherever they did not cover a pixel, so
  the master's background tracked coverage and seams between frame footprints
  showed as broad diagonal bands. On 500 real frames the background went from a
  31→72 ADU ramp to flat, and the star count from 412 to 1001.

## 1.5.4

- Fixed a graphics-memory leak of about 540 MiB per stack. Enough stacks in one
  session exhausted the card and stalled or crashed the machine.

## 1.5.3

- Abort works during frame analysis, not only during stacking.

## 1.5.2

- Reverted the 1.5.1 trail cap: it removed a real, visible trail on some frames.

## 1.5.1

- Trail rejection rolls back implausibly large masks.

## 1.5.0

- Trail rejection stopped over-flagging: the minimum trail length scales with
  the sensor instead of being a flat 15 pixels.

## 1.4.1

- Clear message instead of a silent crash on CPUs without AVX2.
- Thermal governor acts before the GPU throttles rather than after.

## 1.4.0

- Registration now runs while the GPU is still working on later frames, instead
  of waiting for the whole batch.

## 1.3.0

- Faster cosmetic correction, drizzle and trail patching. 200 frames, full
  quality options: 2m00 → 1m41, with an identical master.

## 1.2.0

- Trails are detected at any angle, and the detection runs on the GPU.
- PSF-signal frame weighting.

## 1.1.9

- Abort button. An aborted run keeps its checkpoint, so Resume continues it.
- Darks and flats can be given as folders.
