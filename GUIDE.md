# FS-CUDA — User manual

*(Version française : [aller au manuel en français](#fs-cuda--manuel-utilisateur))*

> ## 💛 Support FS-CUDA — donate from **2 EUR**
> ### 👉 **https://paypal.me/Antoine8082**
>
> FS-CUDA is free to try for **10 full exports**. A donation of **at least
> 2 EUR** unlocks it permanently on your machine: no startup screen, no
> watermark, and the full Phase-2 quality engine (Normalize, Autocrop, RGB
> align, per-pixel weighting, Reject trails, Drizzle).
> After donating, open **Enter license**, copy your **Machine ID**, and send it
> with your PayPal email to **fscuda8082@gmail.com** to receive your key.
>
> *It is written and maintained by one person — if it saves you time, please
> support it.*
>
> ---
>
> ## 💛 Soutenir FS-CUDA — don à partir de **2 EUR**
> ### 👉 **https://paypal.me/Antoine8082**
>
> FS-CUDA est libre d'essai pendant **10 exports complets**. Un don d'**au moins
> 2 EUR** le déverrouille définitivement sur votre machine : plus d'écran de
> démarrage, plus de filigrane, et le moteur de qualité Phase 2 au complet
> (Normalize, Autocrop, RGB align, pondération par pixel, Reject trails,
> Drizzle).
> Après votre don, ouvrez **Enter license**, copiez votre **Machine ID** et
> envoyez-le avec l'adresse e-mail de votre PayPal à **fscuda8082@gmail.com**
> pour recevoir votre clé.
>
> *Il est écrit et maintenu par une seule personne — s'il vous fait gagner du
> temps, merci de le soutenir.*

> **TL;DR — safe defaults for deep-sky OSC:**
> **Debayer = RCD · Rejection = Winsorized Sigma · Weighting = Noise (1/var),**
> with every repair option ticked. These are already the defaults. Change them
> only for the specific reasons below.

---

## 1. How FS-CUDA works

You give it raw colour subs; it gives you one deep, clean, **linear 32-bit
master**. Between those two points a fixed pipeline runs, and each option in the
window switches a stage of it on or off:

1. **Calibrate** — subtract the master dark, divide by the master flat.
2. **Cosmetic / defect repair** — hot and cold pixels, dead rows and columns.
3. **Debayer** — turn the colour mosaic into RGB (skipped in Bayer drizzle).
4. **Measure** — detect the stars in every frame and score its sharpness,
   roundness and noise. This picks the reference frame and drives the weighting.
5. **Register** — match stars between frames and align each onto the reference
   with sub-pixel accuracy.
6. **Normalize** — match every frame's sky background to the reference.
7. **Reject trails** — find satellite and aircraft trails on each registered
   frame and mark those pixels, before anything is combined.
8. **Reject and integrate** — combine the frames, discarding per-pixel outliers
   (cosmic rays, remaining hits) as it averages.
9. **Post** — autocrop, RGB alignment, optional drizzle.

The chain runs on your GPU wherever that is faster, and streams frames through
memory, so the number of frames is not limited by your RAM.

**The master is linear and unstretched.** The bright preview on screen is a
display stretch only — the saved file keeps the full dynamic range for
processing afterwards.

---

## 2. Getting started

### Preparing your frames

- **Lights** — your subs of the target. Raw CFA FITS (`.fits` / `.fit`) straight
  from the camera. **Do not debayer them first.**
- **Darks** *(optional)* — same exposure, gain and temperature as the lights.
- **Flats** *(optional)* — for vignetting and dust shadows.

Each of the three accepts **either a folder of raw frames** — FS-CUDA builds the
master for you — **or a single master FITS** you already made.

**Monochrome camera?** Tick **Mono / LRGB frames** and the frames are taken as
single-channel, with no debayering. One press can stack every filter of a session
and combine them into colour — see *Monochrome cameras and LRGB* in §3.

> **Speed tip:** put the **Output** folder on a *different physical disk* from
> the lights. Large stacks write working data there, so splitting reads and
> writes across two drives is noticeably faster.

### Your first stack

1. **Lights → Open** — pick the folder of subs. Add **Darks** and **Flats** the
   same way if you have them.
2. **Output → Open** — choose where the master goes; set a **Name**
   (e.g. `master.fits`).
3. Leave every option at its default for the first run.
4. Press **Stack**. A progress bar appears with the current stage beneath it.
5. When it finishes the master appears on the right and **Result** shows the
   statistics. Press **Save FITS** to write the file.

**Nothing reaches the disk until you press Save FITS.**

---

## 3. Every option explained

### Checkboxes

**Cosmetic repair** — replaces pixels that differ wildly from their same-colour
neighbours (hot, cold, cosmic-ray hits). *Leave on.* Turn it off only if you
suspect it is eating real, very small stars on an undersampled setup.

**Normalize** — measures each frame's sky background and matches it to the
reference before averaging. Without it, a frame shot through light pollution or
moonlight drags a gradient into the master and corrupts the outlier rejection.
*Leave on.*

**Autocrop** — registered frames never overlap perfectly, leaving a ragged
low-coverage border. This trims to the fully covered rectangle. *Leave on;* turn
it off to keep the whole field including edges.

**RGB align** — corrects small shifts between the red, green and blue channels,
mostly from atmospheric dispersion at low altitude. *Leave on.*

**Defect fix** — repairs entire dead or hot columns and rows, which cosmetic
repair cannot fix because the whole neighbourhood is bad. *Leave on.*

**Reject trails** — finds satellite and aircraft trails and removes them. It
looks for bright straight lines **at any angle** and **masks just those pixels**:
the affected samples are dropped from that one frame and the other frames supply
those pixels instead. Nothing is invented, and **no frame is lost** — on a real
38-frame set this discarded 18,608 pixels instead of three whole subs, about
2,600× less data for the same trails removed, and recovered 11 stars in the
master. *Leave on.*

Four things keep it from removing what you want to keep:

- **Diffraction spikes are spared.** A spike radiates *from a star*, a satellite
  does not, so lines passing through a bright star are left alone. Brightness
  could never make this call — on real frames the spike from a star just outside
  the field was the *strongest* line in most subs, brighter than a genuine
  satellite elsewhere.
- **The width test follows your seeing.** A trail is blurred by the same optics
  as the stars, so it can never be thinner than they are; the limit scales with
  the measured FWHM rather than being fixed.
- **A real object, present in every frame, is never touched** — it cancels
  against the reference and is never even considered.
- **Dithering is not mistaken for a trail.** Every sub stops covering the
  reference somewhere, and that edge is a dead straight line across the whole
  frame — which is exactly what a satellite looks like. Frames used to be
  flagged for their own registration border; the edge and a margin around it are
  now excluded.

After a stack the result panel reports how many frames carried a trail and how
many pixels were masked. Nothing is dropped, so the "Rejected" count stays at
zero; that line is the one that tells you trail rejection did anything. If the
number looks large for your sky, inspect the master along those lines before
trusting it.

**Per-pixel weight** — instead of one weight per frame, weights each pixel by its
own expected noise (from sensor gain and read noise). Slightly deeper faint
detail. *Leave on.*

**Drizzle 2x** — reconstructs on a 2× finer grid by dropping each input pixel
onto the output canvas instead of interpolating. **Only works if your subs were
dithered.** See §4.

**Bayer** — only meaningful with Drizzle. Drops the *raw sensor samples* onto the
canvas with **no debayer interpolation at all**, so no invented colour enters the
master. The sharpest result FS-CUDA can produce. Use it whenever you drizzle
dithered data.

**Mono / LRGB frames** — your camera has no colour filter array, so nothing is
demosaiced and the master stays a single channel at full sensor resolution. Ticking
it reveals the LRGB batch options. See the next section.

**Plate solve** — writes real sky coordinates (WCS) into the saved file. Runs at
**save** time. See §6.

**Resume after interruption** — journals progress so a crash, a power cut or
**Abort** restarts from where it stopped instead of from scratch. Costs almost
nothing. *Leave on.*

### Monochrome cameras and LRGB

Tick **Mono / LRGB frames** for a monochrome camera with a filter wheel. Nothing
is demosaiced, the master keeps one channel at full sensor resolution, and the
Debayer and Bayer-pattern controls disappear because they mean nothing here.
Cosmetic repair automatically switches to comparing each pixel with its eight
immediate neighbours instead of the same-colour neighbours a mosaic requires.

Two further options appear.

**LRGB batch** stacks every filter in one press. You get a row for each; leave a
row empty for a filter you did not shoot.

Red, green and blue are then **combined into one colour master automatically** —
there is no option, because the combine needs all three and tells you when they
are not there. That colour master is what **Save** writes, under the **Name** you
chose.

**Nothing is written to disk until you press Save.** The batch keeps its masters
in memory, so a run you decide to throw away costs you nothing and no half-made
file is left behind.

Narrowband **stacks** but does not take part in the combine, which needs red,
green and blue. That is deliberate: SHO and HOO are conventions rather than
facts, and there is no single correct mapping from three narrowband filters to
three colour channels. Choosing one here would bury a decision you cannot see.
You get the masters; the palette is your processing software's job.

Tell it how your frames are organised:

- **A folder per filter** — one Lights folder and one Flat per filter. Flats are
  per filter because dust shadows and vignetting move with the filter. Leave a
  row empty for a filter you did not shoot.

  **Darks Light** serves L, R, G and B, which are normally the same exposure.
  **Darks Ha**, **Darks OIII** and **Darks SII** are separate because narrowband
  usually is not: 300 s or 600 s subs against 120 s for broadband is ordinary,
  and a dark only cancels the sensor's signal for the *same* exposure. Using a
  120 s dark on a 300 s sub leaves most of the dark current in the master.
  Leave a narrowband Darks row empty and that filter falls back to Darks Light.
- **One folder, split by FITS FILTER header** — for a session captured straight
  into a single directory. Each frame's `FILTER` card decides its channel, and
  the folder is searched recursively, so per-filter subfolders work here too.
  Frames marked as darks, flats or bias by `IMAGETYP` are skipped, so pointing
  Lights at the whole session folder does not stack your calibration frames.

  Press **Scan filters** to see the split before committing to a long run. It
  reads only headers, so it answers immediately even for hundreds of frames.
  Anything with no recognisable filter is **reported in amber, never guessed** —
  silently treating Ha as red would give you a plausible and wrong image.

The combine registers the finished masters onto a common frame. Luminance is the
reference when present; otherwise green, because it has the most stars and so
gives the weakest channel its best chance of aligning. Channel backgrounds are
matched by offset, the edges where the channels do not overlap are trimmed, and
the per-channel FWHM is reported so you are told when one filter is noticeably
softer than the others.

**What Save writes.** One press writes:

- `<name>.fits` — the colour master, plate-solved if **Plate solve** is on.
- `<name>_L`, `<name>_Ha`, `<name>_OIII`, `<name>_SII` — only the filters the
  colour master *cannot* contain. Luminance is just the registration reference,
  and narrowband never takes part, so these would otherwise be lost.

There is no `_R`, `_G` or `_B` file: that data is already in the colour master,
and writing it twice was pure duplication. If red, green and blue are not all
present there is no combine, and Save writes each filter's own master instead —
so a narrowband-only night still exports everything.

One press counts as **one export** against the trial however many files it
writes, and every file carries the same watermark.

**Astrometry.** The colour master, the SHO palette and the HaRGB master all sit
on the *same* pixel grid, so one plate solution describes all three and they
overlay exactly as layers. The single-filter masters (`_Ha`, `_OIII`, `_SII`,
`_L`) are each stacked against their own reference, so they do **not** share that
grid and carry no WCS — copying it onto them would place every star wrongly while
the header still looked perfectly valid.

### Narrowband in colour: SHO and HaRGB

Two extra colour outputs when the filters are there. Both are **palettes or
blends, not measurements**, so each is written under a name that says exactly
what it is — FS-CUDA never applies one silently.

**`<name>_SHO.fits`** — the Hubble palette: **SII → red, Ha → green, OIII →
blue**. Without SII it falls back to **HOO** (Ha → red, OIII → green *and* blue)
and is named `_HOO` instead, so which mapping you got is never in doubt. Without
OIII neither palette exists and nothing is written.

**The three channels are balanced for you.** Sky backgrounds are matched to a
common zero point, then each channel is linearly rescaled so its signal spans the
same range as the others. Without this a SHO is dominated by whichever line is
strongest — measured on real data, Ha carried **3.5×** the signal of SII and
OIII, which is why an unbalanced SHO comes out uniformly green.

The rescale is `out = a × in + b`: strictly **linear**, so nothing is clipped or
gamma-shifted and everything you do downstream behaves normally. The gains used
are reported in the run log. This is the standard mechanical first step, not a
colour calibration — making red mean *red* still needs photometry, and SCNR to
taste is still yours to apply.

**`<name>_HaRGB.fits`** — your true-colour master deepened with narrowband: **Ha
added to red, OIII added to blue**, green untouched. This keeps what only
broadband gives — real colour, and stars whose colour means something — while
letting narrowband, which is often the larger part of a night, carry the nebula.

The blend is a background-matched **maximum**, not a sum. A star lands in both
filters, so adding them inflates it (10000 + 3000); the maximum leaves it at
whatever the better filter measured. Nebulosity is far brighter in narrowband and
wins on its own, with no ratio to invent.

**`<name>_RGB_SHO.fits`** — broadband **colour** with narrowband **structure**.
The one to reach for when the SHO shows detail your RGB does not.

A luminance is built from *all six* filters, weighted by how well each measured
(inverse variance), and it replaces the colour master's own brightness. Colour,
including star colour, stays broadband. Only the structure changes.

> Why all six and not just the narrowband: narrowband alone was measured on real
> M1 masters and it **wrecks the image**. Replacing the luminance replaces the
> brightness outright, so the Crab took the narrowband's amplitude — about 60 ADU
> above sky against broadband's 360 — and nebula SNR fell from 27 to 3.7. The
> structure survived; the contrast did not. With all six it comes out *better*
> than the plain colour master on every channel: sky noise 12.9 → 11.8 and
> nebula SNR 27.3 → 29.0 in red, and similar in green and blue.

The colour is smoothed slightly before the swap while the luminance stays sharp.
Scaling by a luminance injects its noise into all three channels — that showed as
coloured speckle across the background — and colour carries no fine detail the
eye can resolve, which is the same reason every image codec subsamples it.

Expect a **measured** improvement rather than a dramatic one. SHO looks striking
because it throws the continuum away, and that is also what makes it unnatural;
RGB_SHO keeps the true rendition, so it gains structure without the drama.

A **meridian flip between sessions is handled**. Registration matches star
patterns through any rotation, so narrowband shot on the other side of the
meridian — 180° from your broadband — lands correctly without you doing anything.

**The combined image is linear and registered but NOT colour-calibrated.** Making
red mean *red* needs photometry against a star catalogue; do that in your
processing software.

A filter that fails does not discard the others — losing three good channels
because one had an unreadable frame is the wrong behaviour.

### Debayer — how the colour mosaic becomes RGB

*Not used for monochrome cameras; these controls are hidden when Mono is ticked.*

Your sensor sees one colour per pixel; debayering reconstructs the other two.

- **RCD** *(default, best all-round)* — Ratio-Corrected Demosaicing. Handles star
  edges cleanly and roughly halves the colour fringing around bright stars
  compared with simpler methods. Use this unless you have a reason not to.
- **SuperPixel** — combines each 2×2 Bayer group into one RGB pixel. **No
  interpolation at all**, so no colour artefacts — but the image is **half the
  width and height**. Useful on heavily oversampled setups, or when you want
  guaranteed-honest colour and can spare the resolution.

*Ignored when Bayer drizzle is on, because nothing is debayered in that mode.*

### Rejection — throwing out bad pixels

For each output pixel the stack holds one sample per frame; rejection discards
the ones that disagree before averaging.

- **Winsorized sigma** *(default, all stack sizes)* — robust sigma clipping. It
  pulls extreme values towards the middle before measuring the spread, so a
  bright satellite cannot inflate the very threshold meant to catch it. Works
  from a handful of frames to thousands.
- **GESD** — Generalized Extreme Studentized Deviate. A stricter test, genuinely
  better **only on small stacks (below roughly 25 frames)**. Slower, and no
  advantage on big ones.

### Weighting — how much each frame counts

- **Noise (1/var)** *(default)* — inverse-variance weighting: cleaner frames
  count more. Mathematically optimal for reaching the faintest signal. Best for
  deep, faint targets.
- **PSF signal** — weights by star quality, strongly favouring the sharpest
  frames. Choose it when seeing varied a lot and you would rather have a sharper
  master than the absolute deepest one.

### Frame selection

- **Manual (review)** — opens a table of every frame with its measurements
  (stars, FWHM, roundness) so you can untick the ones you do not want.

---

## 4. Sampling, binning and drizzle

### Check your sampling first

Before reaching for drizzle, work out whether your pixels or your *sky* are the
limit. Two numbers do it:

```
    plate scale  =  206.265 x pixel size (um) / focal length (mm)     arcsec/px
    px per FWHM  =  the FWHM the stack reports, in pixels
```

Nyquist needs about **2 pixels per FWHM**; 3 is comfortable. Above that you are
**oversampled** — you are spreading the same photons over more pixels than the
seeing can fill, and every one of them carries its own read noise.

- **More than ~4 px per FWHM → bin, do not drizzle.** Set **Output binning 2×2**.
  You lose no real resolution and gain about **2× the signal-to-noise per pixel**,
  for free, in one click.
- **Under ~2 px per FWHM → you are undersampled**, and drizzle (with dithered
  subs) genuinely recovers detail.
- **In between → leave both off.**

> Worked example, from a real 1325 mm / 3.8 µm rig: plate scale 0.59″/px, and the
> stack reported FWHM 5.71 px = 3.38″. That is **5.7 px per FWHM — oversampled
> about 1.9×.** Binning 2×2 gives 1.18″/px and 2.9 px per FWHM, still properly
> sampled, at twice the SNR. Drizzling that data would have made it *worse*: more
> pixels, no more detail, and the noise spread thinner.

Drizzle is not a sharpening tool. It only helps when the sensor, not the sky, is
what is limiting you.

### Drizzle: when and how

Drizzle recovers detail a normal average cannot, but it has one hard
requirement: **your subs must be dithered.**

- **Dithered** — the mount deliberately shifted the framing a few pixels between
  subs. Almost all capture software can do this.
- **Not dithered** — every sub lands on the same pixels. Drizzle then has no
  extra information and produces a sparse, patchy master.

With dithered data the best-quality combination is **Drizzle 2x + Bayer**. The
output master is twice the width and height of your sensor.

If you are unsure whether you dithered, stack once with drizzle off and once
with it on and compare — the failure mode is obvious.

---

## 5. While it runs, and reading the result

The **stage** under the progress bar shows what the pipeline is doing:
*Reading & measuring → Registering frames → Rejecting trails → Integrating →
Drizzle accumulate → Autocropping.* The fast engine reports fewer of these,
because it does several of them at once rather than one after another.

**Abort** stops at the next safe point. With *Resume after interruption* on,
pressing **Stack** again continues rather than restarting.

Typical durations at full quality on a modern laptop GPU: about **1½ minutes for
200 frames**, about **14 minutes for 1000+**.

| Result field | Meaning |
|---|---|
| **Stacked / Rejected** | Frames used, and frames dropped as unusable |
| **Stars** | Stars detected in the master — more means deeper |
| **FWHM** | Star width in pixels — lower is sharper |
| **Noise** | Background noise — lower is cleaner |
| **Master** | Final pixel dimensions (2× your sensor when drizzling) |

The **Shadows** and **Midtones** sliders change the on-screen stretch only, never
the saved data. **Reset STF** puts them back.

---

## 6. Plate solving

Plate solving runs **when you press Save FITS**, not during stacking — which is
why it never appears in the progress bar.

1. Tick **Plate solve**.
2. Type the target in **Target** and press **Search** — or leave it blank if your
   FITS headers already contain RA/DEC.
3. Check **Focal (mm)** and **Pixel (um)**. They are filled in from the finished
   master's own header (`FOCALLEN` and `XPIXSZ`, which your capture software
   writes), so normally there is nothing to do here.
4. Press **Save FITS**.

FS-CUDA downloads a Gaia star catalogue for that patch of sky, matches it against
the stars in your master, and writes WCS coordinates into the file. The status
line reports the outcome. The window may pause briefly during the download —
that is normal.

**If Focal or Pixel is wrong, the solve cannot work,** because the search runs at
the plate scale they imply. This is the single most common cause of failure, so
the message says so when it happens.

A solve has to rest on **at least 15 matched stars**. Below that it is refused and
**no coordinates are written at all.** That is deliberate: a fit resting on a
handful of stars reports an excellent error figure no matter where it landed —
six stars define the solution exactly, leaving nothing over to disagree — and a
wrong sky position is worse than none, because every program you open the file in
afterwards will trust it. A good solve on a normal field matches hundreds of
stars; a report of six or seven means the optics are wrong, not that the sky was
difficult.

---

## 7. Updates, licence and settings

- **Check for updates** compares your version with the latest release and updates
  the program in place. **Your licence is preserved.**
- FS-CUDA is free to try for **10 complete exports**, with no time limit. After
  that a short donation screen appears at startup and saved files carry a small
  watermark in their metadata — everything keeps working. **Enter license**
  removes both once you have donated.
- Your folders and every option are remembered between launches.

---

## 8. Three everyday recipes

**Deep and faint** (galaxies, nebulae, many subs)
RCD · Winsorized sigma · Noise (1/var) · all repair options on · drizzle only if
dithered.

**Sharp** (bright targets, variable seeing)
RCD · Winsorized sigma · **PSF signal** · Drizzle 2x + Bayer if dithered.

**A whole LRGB night** (monochrome camera and filter wheel)
Mono / LRGB frames · LRGB batch · Winsorized sigma · Noise (1/var) ·
Cosmetic repair on. Point Lights at your session, press **Scan filters** to check
the split, then Stack once and press **Save**. Colour-calibrate the resulting
master in your processing software.

---

## 9. Troubleshooting

| Problem | Cause and fix |
|---|---|
| *"no FITS lights found"* | The folder holds no `.fits`/`.fit` files, or the path is wrong |
| Master mostly black, or very small | Registration failed — usually too few stars. Check focus and exposure; untick Autocrop to see the whole field |
| Drizzle output sparse or patchy | The subs were not dithered. Untick Drizzle |
| A satellite trail survives | Check **Reject trails** is on; it targets trails present in only one or two frames |
| Very slow, or the disk thrashes | Put **Output** on a different physical disk from the lights |
| Program will not start | Your CPU must support AVX2 (Intel 2013 or newer, AMD 2015 or newer). The message says so |
| Colours look wrong | Check the Bayer pattern matches your camera |
| Seems stuck near the end | Large stacks spend a long time integrating; the stage label shows it is still working |
| *"solution rests on only N stars"* | Focal or Pixel is wrong, so the search ran at the wrong scale. Both are read from the header — if you edited them, clear them and re-stack. No coordinates were written, which is intended |
| Plate solve says *"match failed"* | Same cause. Check Focal and Pixel first; a correct field matches hundreds of stars |
| Another program rejects the solved file (e.g. *"0 samples available"* from a spectrophotometric colour calibration) | The written sky position was wrong, from a version before 1.6.4 or from hand-entered optics. Re-stack with 1.6.4 and check the solve reports hundreds of stars |
| LRGB batch skipped frames | They carry no recognisable `FILTER` header — an unlabelled wheel slot, or a filter this version does not know. **Scan filters** lists how many, in amber. L, R, G, B, Ha, OIII and SII are all recognised, under the usual spellings (`H-alpha`, `O3`, `S2` and so on) |
| Narrowband stacked but no colour image | Expected. The colour combine needs red, green and blue; narrowband gets its own masters and the palette (SHO, HOO, …) is chosen in your processing software |
| No files after an LRGB batch | Expected until you press **Save** — the batch keeps its masters in memory so a run you throw away leaves nothing behind |
| No `_R`, `_G` or `_B` file | Expected. That data is inside the colour master; only what the colour master cannot hold (L, Ha, OIII, SII) is written beside it |
| LRGB batch found nothing | Lights points somewhere with no FITS, or the frames have no `FILTER` card — use *A folder per filter* instead |

---
---

# FS-CUDA — Manuel utilisateur

> **En bref — réglages sûrs pour le ciel profond OSC :**
> **Debayer = RCD · Rejection = Winsorized Sigma · Weighting = Noise (1/var),**
> avec toutes les options de réparation cochées. Ce sont déjà les valeurs par
> défaut. Ne les changez que pour les raisons précises indiquées plus bas.

---

## 1. Comment fonctionne FS-CUDA

Vous fournissez des poses couleur brutes ; il produit un **maître linéaire 32
bits**, profond et propre. Entre les deux, un pipeline fixe s'exécute et chaque
option de la fenêtre active ou désactive l'une de ses étapes :

1. **Calibration** — soustraction du dark maître, division par le flat maître.
2. **Réparation cosmétique / défauts** — pixels chauds et froids, lignes et
   colonnes mortes.
3. **Dématriçage** — conversion de la mosaïque couleur en RVB (ignoré en drizzle
   Bayer).
4. **Mesure** — détection des étoiles de chaque image et notation de son piqué,
   de sa rondeur et de son bruit. C'est ce qui choisit l'image de référence et ce
   qui pilote la pondération.
5. **Alignement** — appariement des étoiles entre images et recalage de chacune
   sur la référence avec une précision inférieure au pixel.
6. **Normalisation** — alignement du fond de ciel de chaque image sur la
   référence.
7. **Rejet et intégration** — combinaison des images en éliminant les valeurs
   aberrantes pixel par pixel (satellites, rayons cosmiques, avions).
8. **Finition** — nettoyage des traînées, rognage automatique, alignement RVB,
   drizzle optionnel.

La chaîne s'exécute sur votre GPU partout où c'est plus rapide, et les images
défilent en flux : le nombre d'images n'est donc pas limité par votre RAM.

**Le maître est linéaire, non étiré.** L'aperçu lumineux à l'écran n'est qu'un
étirement d'affichage — le fichier enregistré conserve toute la dynamique pour
le traitement ultérieur.

---

## 2. Premiers pas

### Préparer vos images

- **Lights** — vos poses sur la cible. FITS CFA bruts (`.fits` / `.fit`) tels que
  sortis de la caméra. **Ne les dématricez pas au préalable.**
- **Darks** *(optionnel)* — même temps de pose, même gain, même température que
  les lights.
- **Flats** *(optionnel)* — pour le vignettage et les poussières.

Chacun des trois accepte **soit un dossier d'images brutes** — FS-CUDA construit
le maître pour vous — **soit un fichier FITS maître** déjà préparé.

**Caméra monochrome ?** Cochez **Mono / LRGB frames** : les poses sont alors
traitées en un seul canal, sans dématriçage. Un seul appui peut empiler tous les
filtres d'une session et les combiner en couleur — voir *Caméras monochromes et
LRVB* au §3.

> **Astuce vitesse :** placez le dossier **Output** sur un *disque physique
> différent* de celui des lights. Les grandes piles y écrivent des données de
> travail ; répartir lectures et écritures sur deux disques est nettement plus
> rapide.

### Votre premier empilement

1. **Lights → Open** — choisissez le dossier de poses. Ajoutez **Darks** et
   **Flats** de la même manière si vous en avez.
2. **Output → Open** — choisissez la destination du maître ; saisissez un
   **Name** (par ex. `master.fits`).
3. Laissez toutes les options par défaut pour le premier essai.
4. Appuyez sur **Stack**. Une barre de progression apparaît, avec l'étape en
   cours en dessous.
5. À la fin, le maître s'affiche à droite et **Result** donne les statistiques.
   Appuyez sur **Save FITS** pour écrire le fichier.

**Rien n'est écrit sur le disque avant d'appuyer sur Save FITS.**

---

## 3. Toutes les options expliquées

### Cases à cocher

**Cosmetic repair** — remplace les pixels très différents de leurs voisins de
même couleur (chauds, froids, impacts de rayons cosmiques). *À laisser activé.*
Ne le désactivez que si vous soupçonnez qu'il efface de vraies très petites
étoiles sur un montage sous-échantillonné.

**Normalize** — mesure le fond de ciel de chaque image et l'aligne sur la
référence avant la moyenne. Sans cela, une image prise sous pollution lumineuse
ou sous la Lune introduit un gradient dans le maître et fausse le rejet des
valeurs aberrantes. *À laisser activé.*

**Autocrop** — une fois alignées, les images ne se recouvrent jamais
parfaitement et laissent une bordure irrégulière peu couverte. Cette option
rogne au rectangle entièrement couvert. *À laisser activé ;* désactivez-la pour
conserver tout le champ, bords compris.

**RGB align** — corrige les petits décalages entre couches rouge, verte et bleue,
dus surtout à la dispersion atmosphérique à basse hauteur. *À laisser activé.*

**Defect fix** — répare les colonnes et lignes entièrement mortes ou chaudes, que
la réparation cosmétique ne peut pas corriger puisque tout le voisinage est
mauvais. *À laisser activé.*

**Reject trails** — repère les traînées de satellites et d'avions et les
supprime. Il cherche des lignes droites brillantes **à n'importe quel angle** et
**masque uniquement ces pixels** : les échantillons concernés sont retirés de
cette seule image et les autres images fournissent ces pixels à la place. Rien
n'est inventé, et **aucune image n'est perdue** — sur un jeu réel de 38 poses,
cela a écarté 18 608 pixels au lieu de trois poses entières, soit environ 2 600
fois moins de données pour les mêmes traînées supprimées, et récupéré 11 étoiles
dans le maître. *À laisser activé.*

Quatre garde-fous l'empêchent de retirer ce que vous voulez garder :

- **Les aigrettes de diffraction sont épargnées.** Une aigrette rayonne *depuis
  une étoile*, pas un satellite : les lignes passant par une étoile brillante
  sont donc laissées intactes. La luminosité ne pouvait pas trancher — sur de
  vraies poses, l'aigrette d'une étoile juste hors champ était la ligne la *plus
  brillante* de la plupart des poses, davantage qu'un vrai satellite ailleurs.
- **Le test de largeur suit votre seeing.** Une traînée est floutée par la même
  optique que les étoiles : elle ne peut pas être plus fine qu'elles. La limite
  s'adapte à la FWHM mesurée au lieu d'être fixe.
- **Un objet réel, présent sur toutes les images, n'est jamais touché** — il
  s'annule face à la référence et n'est même pas envisagé.
- **Le dithering n'est pas pris pour une traînée.** Chaque pose cesse de couvrir
  la référence quelque part, et ce bord est une ligne parfaitement droite en
  travers de toute l'image — exactement ce à quoi ressemble un satellite. Des
  poses étaient auparavant signalées à cause de leur propre bord de recalage ; ce
  bord et une marge autour sont désormais exclus.

Après un empilement, le panneau de résultat indique combien de poses portaient
une traînée et combien de pixels ont été masqués. Rien n'étant supprimé, le
compteur « Rejected » reste à zéro : c'est cette ligne qui vous dit que le rejet
des traînées a fait quelque chose. Si le nombre paraît élevé pour votre ciel,
inspectez le maître le long de ces lignes avant de lui faire confiance.

**Per-pixel weight** — au lieu d'un poids unique par image, pondère chaque pixel
selon son propre bruit attendu (gain du capteur et bruit de lecture). Détails
faibles légèrement plus profonds. *À laisser activé.*

**Drizzle 2x** — reconstruit sur une grille 2× plus fine en déposant chaque pixel
d'entrée sur la toile de sortie au lieu d'interpoler. **Ne fonctionne que si vos
poses ont été dithérées.** Voir §4.

**Bayer** — n'a de sens qu'avec Drizzle. Dépose les *échantillons bruts du
capteur* sur la toile **sans aucune interpolation de dématriçage** : aucune
couleur inventée n'entre dans le maître. Le résultat le plus piqué que FS-CUDA
sache produire. À utiliser dès que vous drizzlez des données dithérées.

**Mono / LRGB frames** — votre caméra n'a pas de matrice de filtres colorés :
rien n'est dématricé et le maître reste sur un seul canal, à la pleine résolution
du capteur. Cocher cette case fait apparaître les options d'empilement LRVB. Voir
la section suivante.

**Plate solve** — écrit les vraies coordonnées célestes (WCS) dans le fichier
enregistré. S'exécute à l'**enregistrement**. Voir §6.

**Resume after interruption** — journalise la progression : un plantage, une
coupure de courant ou **Abort** repartent du point d'arrêt au lieu de tout
recommencer. Coût quasi nul. *À laisser activé.*

### Caméras monochromes et LRVB

Cochez **Mono / LRGB frames** pour une caméra monochrome avec roue à filtres.
Rien n'est dématricé, le maître conserve un seul canal à la pleine résolution du
capteur, et les réglages Debayer et motif de Bayer disparaissent car ils n'ont
plus de sens. La réparation cosmétique passe automatiquement à la comparaison de
chaque pixel avec ses huit voisins immédiats, au lieu des voisins de même couleur
qu'exige une mosaïque.

Deux options supplémentaires apparaissent.

**LRGB batch** empile tous les filtres en une seule fois. Chaque filtre a sa
ligne ; laissez vide celle d'un filtre que vous n'avez pas photographié.

Le rouge, le vert et le bleu sont ensuite **combinés automatiquement en un seul
maître couleur** — il n'y a pas d'option, car la combinaison exige les trois et
vous prévient quand ils ne sont pas tous là. C'est ce maître couleur que **Save**
écrit, sous le **Name** que vous avez choisi.

**Rien n'est écrit sur le disque tant que vous n'avez pas appuyé sur Save.** Le
lot conserve ses maîtres en mémoire : un traitement que vous décidez de jeter ne
vous coûte rien et ne laisse aucun fichier à moitié fait derrière lui.

La bande étroite **s'empile** mais ne participe pas à la combinaison, qui exige
le rouge, le vert et le bleu. C'est délibéré : SHO et HOO sont des conventions,
pas des faits, et il n'existe pas de correspondance unique et correcte entre
trois filtres à bande étroite et trois canaux couleur. En choisir une ici
enfouirait une décision que vous ne pourriez pas voir. Vous obtenez les maîtres ;
la palette relève de votre logiciel de traitement.

Indiquez-lui comment vos fichiers sont organisés :

- **Un dossier par filtre** — un dossier de poses et un flat par filtre. Les flats
  sont propres à chaque filtre, car les poussières et le vignettage le suivent.
  Laissez une ligne vide pour un filtre que vous n'avez pas photographié.

  **Darks Light** sert à L, R, G et B, qui ont normalement le même temps de pose.
  **Darks Ha**, **Darks OIII** et **Darks SII** sont séparés parce que la bande
  étroite, elle, ne l'a normalement pas : 300 s ou 600 s contre 120 s en large
  bande est courant, et un dark ne compense le signal du capteur que pour le
  *même* temps de pose. Utiliser un dark de 120 s sur une pose de 300 s laisse
  l'essentiel du courant d'obscurité dans le maître. Si vous laissez une ligne
  Darks de bande étroite vide, ce filtre retombe sur Darks Light.
- **Un seul dossier, réparti par l'en-tête FITS FILTER** — pour une session
  enregistrée directement dans un seul répertoire. La carte `FILTER` de chaque
  pose détermine son canal, et le dossier est parcouru récursivement : des
  sous-dossiers par filtre fonctionnent donc aussi. Les images marquées dark,
  flat ou bias par `IMAGETYP` sont ignorées, si bien que désigner le dossier
  complet de la session n'empile pas vos images de calibration.

  Appuyez sur **Scan filters** pour vérifier la répartition avant de lancer un
  long traitement. Seuls les en-têtes sont lus : la réponse est immédiate, même
  pour des centaines de poses. Tout filtre non reconnu est **signalé en orange,
  jamais deviné** — traiter silencieusement du Ha comme du rouge donnerait une
  image plausible et fausse.

La combinaison aligne les maîtres obtenus sur une référence commune. La luminance
sert de référence si elle existe ; sinon le vert, car il contient le plus
d'étoiles et donne donc au canal le plus faible sa meilleure chance de s'aligner.
Les fonds de ciel sont ajustés par décalage, les bords où les canaux ne se
recouvrent pas sont rognés, et la FWHM de chaque canal est indiquée : vous êtes
ainsi averti si un filtre est nettement plus flou que les autres.

**Ce que Save écrit.** Une seule pression écrit :

- `<nom>.fits` — le maître couleur, résolu astrométriquement si **Plate solve**
  est actif.
- `<nom>_L`, `<nom>_Ha`, `<nom>_OIII`, `<nom>_SII` — uniquement les filtres que
  le maître couleur *ne peut pas* contenir. La luminance n'est que la référence
  d'alignement et la bande étroite ne participe jamais : sans cela, ces maîtres
  seraient perdus.

Il n'y a plus de fichier `_R`, `_G` ni `_B` : ces données sont déjà dans le
maître couleur, et les écrire deux fois était une pure duplication. Si le rouge,
le vert et le bleu ne sont pas tous présents, il n'y a pas de combinaison et Save
écrit alors le maître de chaque filtre — une nuit entièrement en bande étroite
exporte donc bien tout.

Une pression compte pour **un seul export** sur la période d'essai, quel que soit
le nombre de fichiers écrits, et tous portent le même filigrane.

**Astrométrie.** Le maître couleur, la palette SHO et le maître HaRGB reposent
sur la *même* grille de pixels : une seule solution astrométrique les décrit tous
les trois et ils se superposent exactement en calques. Les maîtres d'un seul
filtre (`_Ha`, `_OIII`, `_SII`, `_L`) sont empilés chacun sur leur propre
référence : ils ne partagent donc pas cette grille et ne portent aucun WCS — y
recopier la solution placerait chaque étoile au mauvais endroit alors que
l'en-tête paraîtrait parfaitement valide.

### La bande étroite en couleur : SHO et HaRGB

Deux sorties couleur supplémentaires quand les filtres sont là. Ce sont des
**palettes ou des mélanges, pas des mesures** : chacune porte donc un nom qui dit
exactement ce qu'elle est — FS-CUDA n'en applique jamais une en silence.

**`<nom>_SHO.fits`** — la palette Hubble : **SII → rouge, Ha → vert, OIII →
bleu**. Sans SII, repli sur **HOO** (Ha → rouge, OIII → vert *et* bleu), nommé
`_HOO` : la correspondance obtenue n'est jamais ambiguë. Sans OIII, aucune des
deux palettes n'existe et rien n'est écrit.

**Les trois canaux sont équilibrés pour vous.** Les fonds de ciel sont ramenés à
un zéro commun, puis chaque canal est redimensionné linéairement pour que son
signal couvre la même plage que les autres. Sans cela, un SHO est dominé par la
raie la plus forte — mesuré sur de vraies données, Ha portait **3,5×** le signal
de SII et OIII, d'où le SHO uniformément vert.

Le redimensionnement est `sortie = a × entrée + b` : strictement **linéaire**,
donc rien n'est écrêté ni modifié en gamma, et tout ce que vous ferez ensuite se
comporte normalement. Les gains employés sont indiqués dans le journal. C'est
l'étape mécanique habituelle, pas un étalonnage couleur : faire que le rouge
signifie *rouge* demande toujours de la photométrie, et le SCNR reste à votre
convenance.

**`<nom>_HaRGB.fits`** — votre maître en vraies couleurs approfondi par la bande
étroite : **Ha ajouté au rouge, OIII ajouté au bleu**, vert intact. On conserve
ce que seule la large bande donne — les vraies couleurs, et des étoiles dont la
couleur a un sens — pendant que la bande étroite, souvent la plus grande part
d'une nuit, porte la nébuleuse.

Le mélange est un **maximum** après ajustement du fond, pas une somme. Une étoile
est présente dans les deux filtres : les additionner la gonfle (10000 + 3000),
alors que le maximum la laisse à la valeur mesurée par le meilleur filtre. La
nébulosité est bien plus brillante en bande étroite et l'emporte d'elle-même,
sans aucun rapport à inventer.

**`<nom>_RGB_SHO.fits`** — les **couleurs** de la large bande avec la
**structure** de la bande étroite. À utiliser quand le SHO montre des détails que
votre RVB ne montre pas.

Une luminance est construite à partir des *six* filtres, pondérés selon la
qualité de leur mesure (variance inverse), et elle remplace la luminosité du
maître couleur. Les couleurs, y compris celles des étoiles, restent celles de la
large bande. Seule la structure change.

> Pourquoi les six et pas seulement la bande étroite : la bande étroite seule a
> été mesurée sur de vrais maîtres M1 et elle **détruit l'image**. Remplacer la
> luminance remplace la luminosité elle-même : le Crabe a pris l'amplitude de la
> bande étroite — environ 60 ADU au-dessus du ciel contre 360 en large bande — et
> le RSB de la nébuleuse est tombé de 27 à 3,7. La structure survivait, le
> contraste non. Avec les six, le résultat est *meilleur* que le maître couleur
> sur chaque canal : bruit de ciel 12,9 → 11,8 et RSB 27,3 → 29,0 dans le rouge.

Les couleurs sont légèrement lissées avant l'échange, la luminance restant nette.
Multiplier par une luminance injecte son bruit dans les trois canaux — cela se
voyait comme un moucheté coloré sur le fond — et la couleur ne porte aucun détail
fin que l'œil puisse résoudre, pour la même raison que tous les codecs d'image la
sous-échantillonnent.

Attendez-vous à une amélioration **mesurée** plutôt que spectaculaire. Le SHO
frappe parce qu'il jette le continuum, ce qui le rend aussi peu naturel ;
RGB_SHO conserve le rendu réel et gagne en structure sans l'effet de choc.

Un **retournement au méridien entre les sessions est géré**. L'alignement met en
correspondance des motifs d'étoiles quelle que soit la rotation : une bande
étroite prise de l'autre côté du méridien — à 180° de votre large bande — se
recale correctement sans rien faire de votre part.

**L'image combinée est linéaire et alignée mais PAS étalonnée en couleur.** Pour
que le rouge signifie vraiment le rouge, il faut une photométrie sur un catalogue
d'étoiles : faites-le dans votre logiciel de traitement.

Un filtre en échec n'annule pas les autres — perdre trois bons canaux parce qu'un
seul contenait une image illisible serait le mauvais comportement.

### Debayer — de la mosaïque couleur au RVB

*Inutilisé pour les caméras monochromes ; ces réglages sont masqués lorsque Mono
est coché.*

Votre capteur ne voit qu'une couleur par pixel ; le dématriçage reconstitue les
deux autres.

- **RCD** *(défaut, meilleur compromis)* — Ratio-Corrected Demosaicing. Traite
  proprement les bords d'étoiles et réduit d'environ moitié les franges colorées
  autour des étoiles brillantes par rapport aux méthodes plus simples. À utiliser
  sauf raison contraire.
- **SuperPixel** — combine chaque groupe de Bayer 2×2 en un pixel RVB. **Aucune
  interpolation**, donc aucun artefact de couleur — mais l'image fait **la moitié
  de la largeur et de la hauteur**. Utile sur les montages très
  sur-échantillonnés, ou quand vous voulez une couleur garantie sans invention et
  pouvez sacrifier de la résolution.

*Ignoré lorsque le drizzle Bayer est actif, puisque rien n'est dématricé dans ce
mode.*

### Rejection — éliminer les mauvais pixels

Pour chaque pixel de sortie, la pile contient un échantillon par image ; le rejet
écarte ceux qui divergent avant la moyenne.

- **Winsorized sigma** *(défaut, toutes tailles de pile)* — écrêtage sigma
  robuste. Ramène les valeurs extrêmes vers le centre avant de mesurer la
  dispersion, de sorte qu'un satellite brillant ne puisse pas gonfler le seuil
  censé le détecter. Efficace de quelques images à plusieurs milliers.
- **GESD** — Generalized Extreme Studentized Deviate. Test plus strict,
  réellement meilleur **uniquement sur les petites piles (moins d'environ 25
  images)**. Plus lent, sans avantage sur les grandes.

### Weighting — le poids de chaque image

- **Noise (1/var)** *(défaut)* — pondération par l'inverse de la variance : les
  images les plus propres comptent davantage. Optimal mathématiquement pour
  atteindre le signal le plus faible. Idéal pour les cibles profondes et faibles.
- **PSF signal** — pondère selon la qualité des étoiles, en favorisant fortement
  les images les plus piquées. À choisir quand le seeing a beaucoup varié et que
  vous préférez un maître plus piqué au maître le plus profond possible.

### Frame selection

- **Manual (review)** — ouvre un tableau de toutes les images avec leurs mesures
  (étoiles, FWHM, rondeur) pour décocher celles que vous ne voulez pas.

---

## 4. Échantillonnage, binning et drizzle

### Vérifiez d'abord votre échantillonnage

Avant de vous tourner vers le drizzle, déterminez si la limite vient de vos
pixels ou du *ciel*. Deux nombres suffisent :

```
    échelle      =  206,265 x taille pixel (um) / focale (mm)     arcsec/px
    px par FWHM  =  la FWHM annoncée par l'empilement, en pixels
```

Nyquist demande environ **2 pixels par FWHM** ; 3 est confortable. Au-delà, vous
**suréchantillonnez** : les mêmes photons s'étalent sur plus de pixels que le
seeing ne peut en remplir, et chacun apporte son propre bruit de lecture.

- **Plus d'environ 4 px par FWHM → binnez, ne drizzlez pas.** Activez **Output
  binning 2×2**. Vous ne perdez aucune résolution réelle et gagnez environ **2×
  le rapport signal/bruit par pixel**, gratuitement, en un clic.
- **Moins d'environ 2 px par FWHM → vous sous-échantillonnez**, et le drizzle
  (avec des poses dithérées) récupère réellement du détail.
- **Entre les deux → laissez les deux désactivés.**

> Exemple réel, monture 1325 mm / 3,8 µm : échelle 0,59″/px et FWHM annoncée de
> 5,71 px = 3,38″. Soit **5,7 px par FWHM — suréchantillonné environ 1,9×.** Un
> binning 2×2 donne 1,18″/px et 2,9 px par FWHM, toujours correctement
> échantillonné, avec deux fois le RSB. Drizzler ces données les aurait
> *dégradées* : plus de pixels, pas plus de détail, et le bruit étalé plus mince.

Le drizzle n'est pas un outil de netteté. Il n'aide que lorsque c'est le capteur,
et non le ciel, qui vous limite.

### Le drizzle : quand et comment

Le drizzle récupère des détails qu'une moyenne classique ne restitue pas, mais il
impose une condition stricte : **vos poses doivent être dithérées.**

- **Dithérées** — la monture a décalé volontairement le cadrage de quelques
  pixels entre les poses. La quasi-totalité des logiciels d'acquisition sait le
  faire.
- **Non dithérées** — chaque pose tombe sur les mêmes pixels. Le drizzle n'a
  alors aucune information supplémentaire et produit un maître clairsemé et
  troué.

Avec des données dithérées, la meilleure combinaison est **Drizzle 2x + Bayer**.
Le maître de sortie fait deux fois la largeur et la hauteur du capteur.

En cas de doute, empilez une fois sans drizzle et une fois avec, puis comparez :
l'échec est visible immédiatement.

---

## 5. Pendant le traitement, et lecture du résultat

L'**étape** sous la barre de progression indique ce que fait le pipeline :
*Reading & measuring → Registering frames → Rejecting trails → Integrating →
Drizzle accumulate → Autocropping.* Le moteur rapide en affiche moins, car il en
mène plusieurs de front au lieu de les enchaîner.

**Abort** arrête au prochain point sûr. Avec *Resume after interruption* activé,
appuyer de nouveau sur **Stack** reprend au lieu de tout recommencer.

Durées typiques en qualité maximale sur un GPU portable récent : environ **1 min
30 pour 200 images**, environ **14 minutes au-delà de 1000**.

| Champ de Result | Signification |
|---|---|
| **Stacked / Rejected** | Images utilisées, et images écartées comme inutilisables |
| **Stars** | Étoiles détectées dans le maître — davantage = plus profond |
| **FWHM** | Largeur des étoiles en pixels — plus bas = plus piqué |
| **Noise** | Bruit de fond — plus bas = plus propre |
| **Master** | Dimensions finales en pixels (2× le capteur en drizzle) |

Les curseurs **Shadows** et **Midtones** ne modifient que l'étirement d'écran,
jamais les données enregistrées. **Reset STF** les réinitialise.

---

## 6. Astrométrie (plate solve)

L'astrométrie s'exécute **au moment où vous appuyez sur Save FITS**, et non
pendant l'empilement — c'est pourquoi elle n'apparaît jamais dans la barre de
progression.

1. Cochez **Plate solve**.
2. Saisissez la cible dans **Target** et appuyez sur **Search** — ou laissez vide
   si vos en-têtes FITS contiennent déjà RA/DEC.
3. Vérifiez **Focal (mm)** et **Pixel (um)**. Ces valeurs sont reprises de
   l'en-tête du maître lui-même (`FOCALLEN` et `XPIXSZ`, écrits par votre logiciel
   d'acquisition) : normalement, il n'y a rien à faire ici.
4. Appuyez sur **Save FITS**.

FS-CUDA télécharge un catalogue d'étoiles Gaia pour cette portion de ciel, le
compare aux étoiles de votre maître et écrit les coordonnées WCS dans le fichier.
La ligne d'état indique le résultat. La fenêtre peut se figer brièvement pendant
le téléchargement : c'est normal.

**Si Focal ou Pixel est incorrect, l'astrométrie ne peut pas aboutir,** car la
recherche s'effectue à l'échelle de plaque qu'ils impliquent. C'est de loin la
cause d'échec la plus fréquente : le message le signale donc explicitement.

Une solution doit reposer sur **au moins 15 étoiles appariées**. En dessous, elle
est refusée et **aucune coordonnée n'est écrite.** C'est délibéré : un ajustement
reposant sur une poignée d'étoiles affiche une excellente erreur quel que soit
l'endroit où il a atterri — six étoiles définissent exactement la solution, sans
rien laisser pour la contredire — et une position céleste fausse est pire
qu'aucune, car tous les logiciels qui ouvriront ensuite le fichier lui feront
confiance. Une bonne solution sur un champ normal apparie des centaines
d'étoiles ; un résultat à six ou sept signifie que l'optique est mal renseignée,
non que le ciel était difficile.

---

## 7. Mises à jour, licence et réglages

- **Check for updates** compare votre version à la dernière publiée et met à jour
  le programme sur place. **Votre licence est conservée.**
- FS-CUDA est libre d'essai pendant **10 exports complets**, sans limite de
  durée. Ensuite, un court écran de don apparaît au démarrage et les fichiers
  enregistrés portent un petit filigrane dans leurs métadonnées — tout continue
  de fonctionner. **Enter license** supprime les deux une fois votre don
  effectué.
- Vos dossiers et toutes vos options sont mémorisés d'une session à l'autre.

---

## 8. Trois recettes quotidiennes

**Profond et faible** (galaxies, nébuleuses, beaucoup de poses)
RCD · Winsorized sigma · Noise (1/var) · toutes les réparations activées ·
drizzle uniquement si dithéré.

**Piqué** (cibles brillantes, seeing variable)
RCD · Winsorized sigma · **PSF signal** · Drizzle 2x + Bayer si dithéré.

**Une nuit LRVB complète** (caméra monochrome et roue à filtres)
Mono / LRGB frames · LRGB batch · Winsorized sigma · Noise (1/var) ·
Cosmetic repair activé. Désignez votre session dans Lights, appuyez sur **Scan
filters** pour vérifier la répartition, empilez une seule fois puis appuyez sur
**Save**. Étalonnez ensuite la couleur du maître obtenu dans votre logiciel de
traitement.

---

## 9. Dépannage

| Problème | Cause et solution |
|---|---|
| *« no FITS lights found »* | Le dossier ne contient aucun `.fits`/`.fit`, ou le chemin est incorrect |
| Maître presque noir, ou très petit | L'alignement a échoué — en général trop peu d'étoiles. Vérifiez mise au point et pose ; décochez Autocrop pour voir tout le champ |
| Résultat drizzle clairsemé ou troué | Les poses n'étaient pas dithérées. Décochez Drizzle |
| Une traînée de satellite subsiste | Vérifiez que **Reject trails** est actif ; il cible les traînées présentes sur une ou deux images seulement |
| Très lent, ou disque saturé | Placez **Output** sur un disque physique différent de celui des lights |
| Le programme ne démarre pas | Votre processeur doit gérer l'AVX2 (Intel 2013 ou plus récent, AMD 2015 ou plus récent). Le message le précise |
| Couleurs incorrectes | Vérifiez que la matrice de Bayer correspond à votre caméra |
| Semble bloqué vers la fin | Les grandes piles passent longtemps dans l'intégration ; le nom de l'étape montre qu'il travaille encore |
| *« solution rests on only N stars »* | Focal ou Pixel est incorrect : la recherche s'est faite à la mauvaise échelle. Les deux sont lus dans l'en-tête — si vous les avez modifiés, effacez-les et réempilez. Aucune coordonnée n'a été écrite, c'est voulu |
| L'astrométrie affiche *« match failed »* | Même cause. Vérifiez d'abord Focal et Pixel ; un champ correct apparie des centaines d'étoiles |
| Un autre logiciel rejette le fichier résolu (ex. *« 0 samples available »* lors d'un étalonnage couleur spectrophotométrique) | La position céleste écrite était fausse, par une version antérieure à la 1.6.4 ou par une optique saisie à la main. Réempilez avec la 1.6.4 et vérifiez que la solution annonce des centaines d'étoiles |
| L'empilement LRVB a ignoré des poses | Elles ne portent pas d'en-tête `FILTER` exploitable — un emplacement de roue non nommé, ou un filtre que cette version ne connaît pas. **Scan filters** en indique le nombre, en orange. L, R, G, B, Ha, OIII et SII sont tous reconnus, sous les orthographes usuelles (`H-alpha`, `O3`, `S2`, etc.) |
| Bande étroite empilée mais pas d'image couleur | C'est normal. La combinaison couleur exige le rouge, le vert et le bleu ; la bande étroite produit ses propres maîtres et la palette (SHO, HOO, …) se choisit dans votre logiciel de traitement |
| Aucun fichier après un lot LRGB | Normal tant que vous n'avez pas appuyé sur **Save** — le lot garde ses maîtres en mémoire, donc un traitement que vous jetez ne laisse rien |
| Pas de fichier `_R`, `_G` ni `_B` | Normal. Ces données sont dans le maître couleur ; seul ce qu'il ne peut pas contenir (L, Ha, OIII, SII) est écrit à côté |
| L'empilement LRVB n'a rien trouvé | Lights désigne un endroit sans FITS, ou les poses n'ont pas de carte `FILTER` — utilisez plutôt *Un dossier par filtre* |
