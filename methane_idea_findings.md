# The methane idea: what we found

*Summary of our investigation on 17 November 2026: from Darius's idea and Zsolt's reply, through the NASA archive, to what the data actually looks like. About a 10-minute read.*

---

## 1. Where it started: Darius's idea

On 12 November, Darius proposed adding **a second kind of image** to the normal colour (RGB) JunoCam images:

- JunoCam has a **near-infrared filter at about 889 nm**, just beyond what the eye can see.
- At that wavelength, **methane in Jupiter's atmosphere absorbs sunlight**, so clouds stand out more and cloud-free areas look darker.
- He found **no paper that combines RGB with this band**, and he found an architecture with **two encoders** that could help: one branch reads the RGB image, the other reads the methane image.
- **His worry:** we don't have a dataset of **pairs**, meaning an RGB image and a methane image of the same place at the same moment. We only have them separately.

*(One small wording fix: it isn't a **thermal** band. 889 nm is reflected sunlight, not heat that Jupiter gives off.)*

## 2. What our mentor said

Zsolt replied in short: **"You don't need a separate dataset. Build it yourselves from the raw data."** His reasoning: JunoCam has four filters (blue, green, red and methane) **on the same sensor**. He also sent us links and three warnings.

**Warning 1: the methane filter doesn't measure how much methane there is.** It measures **how much sunlight comes back** in a colour that methane absorbs. Methane is spread fairly evenly through the atmosphere, so what changes from place to place is **how much methane the light had to travel through**:
- **high clouds:** the light bounces back early and little is absorbed, so they look **bright**;
- **low, deep clouds:** the light passes through a lot of methane and gets absorbed, so they look **dark**.

So a methane image is really a **cloud-height (and high-haze) map**. He suggests framing the project as *"estimating cloud height/opacity from RGB"*: the RGB image is the input, and the methane image is what the model learns to predict.

**Warning 2: the methane band isn't in every image.** It needs very long exposures and was designed mainly for the **poles**. Check which orbits have it first.

**Warning 3: use calibrated data (RDR), not the PNGs from the mission website**, which are only 8-bit.

**On the model:** start with a **simple baseline** (a small U-Net), and only then try the two-encoder architecture, because the dataset will be small.

### A few terms we had to clear up

- **Encoder:** the part of a neural network that takes an image and compresses it into *features*, a summary of edges, shapes and textures. "Two encoders" = one per input type, combined later.
- **U-Net / baseline:** a U-Net is a standard model that takes an image in and gives an image out. The *baseline* is the simple model the fancy one must beat. **Beating it is our research result.**
- **Orbit / perijove (PJ):** Juno moves around Jupiter in a long loop and only gets close briefly each time. That close pass is the *perijove*, when JunoCam takes its photos. "Orbit 21" or "PJ21" = the 21st close pass = one batch of images.
- **EDR vs RDR:** two versions of every image in NASA's archive. **EDR** is raw, exactly as the camera sent it. **RDR** is **calibrated**: its pixel values correspond to real amounts of light.

## 3. How this relates to what we already have

*Background: Rareș has a test notebook (`Rospin/teste/test.ipynb`, not in this repo yet). It loads the Jovian Vortex Hunter data and draws each labelled vortex's ellipse on its image crop. The ellipses' `x`, `y` values turned out not to be image positions, so the notebook places each ellipse from its latitude/longitude instead (median error ~6 px on 384-px crops) and saves one clean label per crop and vortex (`crop_vortex_labels.csv`).*

That notebook currently uses **two sources** (mentioned for the original idea), neither of which is the NASA archive:
- **the images:** 68,322 ready-made 384 × 384 crops from **Hugging Face** (`ramanakumars/JovianVortexHunter`);
- **the labels:** the Jovian Vortex Hunter vortex catalogue (the JSON) from **Zenodo**.

Those crops are **RGB only**, with no methane band. So for this idea we have to go back to the original JunoCam data.

**Do the methane images have to come from the same orbits as our labels?** Only if we want to **combine methane with our vortex labels**. Our JSON lists perijoves **13 to 36**, and the labels only describe the clouds of those passes; Jupiter's clouds move and change. A nice combined question would be *"are some vortices (for example red ones) higher than others?"*: look up each labelled vortex in the methane image.

If we do Darius's idea **on its own** (predict methane from RGB), **any orbit with methane images works**, and we don't need the labels at all.

## 4. The links, one by one

### Link 1: the official NASA archive (PDS)
`https://pds-imaging.jpl.nasa.gov/volumes/juno.html`

This is where all JunoCam data lives. It's split into numbered packages called **volumes** (`JNOJNC_0001` to `JNOJNC_0035`), each covering one to four orbits. **Our perijoves 13–36 are in volumes `JNOJNC_0008` to `JNOJNC_0019`**, two orbits each. No account is needed, and everything is free.

**How to access it:** click a volume, go into `DATA/`, then `EDR/` or `RDR/`, then `JUPITER/`, then `ORBIT_XX/`, and download an image (`.IMG`) **together with** its description file (`.LBL`). It can also be scripted with Python `requests` or `wget`. Each volume also has an `INDEX/INDEX.TAB` file listing every image. That's how we did all the counting below without downloading thousands of images.

### Link 2: an example folder
`https://planetarydata.jpl.nasa.gov/img/data/juno/JNOJNC_0028/DATA/EDR/JUPITER/`

This is one concrete folder of raw images (orbits 57–59), showing what Link 1 leads to.

### Link 3: the documentation of the files
`.../JNOJNC_0024/DOCUMENT/JUNO_JNC_EDR_RDR_DPSIS.HTM`

This is the official description of the file format, and it answered several of our questions.

- **The camera:** one 1600 × 1200 sensor with **four colour strips glued directly on it**: red, green, blue and methane (≈890 nm). Juno spins, so each strip photographs the same piece of Jupiter in turn. An image is made of thin slices called **framelets** (1648 × 128 pixels), one per filter per moment, **stacked in time order** (moment 1 blue, green, red, …, then moment 2, …). That's why a raw image looks like a pile of stripes rather than a picture.
- **The file name tells you the filter.** For example `JNCE_2018197_14M00024_V01.IMG`: `E` = raw (`R` = calibrated), `2018197` = year and day, `14` = orbit, and **`M` = filter**. The filter letters are:
  - **A** = all four filters, including methane;
  - **M** = methane only;
  - **C** or **P** = RGB;
  - **H** = methane **global map**.
- **Why calibration is needed:** the camera measures with 12-bit precision (4,096 levels) but **squeezes the values to 8 bits** before sending them, to save bandwidth. The raw numbers also depend on exposure time and filter sensitivity. The **RDR** reverses the squeeze and **scales values so a perfectly white surface would read 10,000**, which makes images comparable. RDR files are 16-bit.
- **No geometry correction:** the images are *not* corrected for Juno's motion. To know which pixel shows which point on Jupiter, you need **SPICE** (NASA's files describing where the spacecraft was and where it pointed) plus a projection tool. *(SPICE is the data, ISIS is the tool that uses it, so we need both.)*
- **Lost data** appears as rows of **zeros**. Treat them as "missing", not as black.
- **Folders:** `DATA/EDR`, `DATA/RDR`, **`DATA/GLOBAL_MAPS`** (ready-projected maps of the whole planet), `INDEX/` and `EXTRAS/` (timing corrections).

### Link 5: Brian Swift's code
`https://github.com/BrianSwift/JunoCam`

We cloned it and looked through it (about 70 files, ~650 MB). It's a community pipeline that turns raw images into **3D models for rendering beautiful pictures in Blender**.
- **Bad:** it's written entirely in **Mathematica**, which is paid software. The author himself writes that these are *"not production quality tools"*. Most importantly, **methane isn't actually processed**: it appears only in a list of filter names and in a piece of code that is **commented out**.
- **Good:** it contains useful pieces, such as brightness corrections (flat fields), a lens model of the camera, and a template for loading SPICE.
- **Verdict:** use it as a reference, not as a tool.

### Link 6: ISIS `junocam2isis`
`https://isis.astrogeology.usgs.gov/.../junocam2isis.html`

**ISIS** is a free toolkit from the US Geological Survey for processing images from planetary missions. It's hundreds of small command-line programs, each doing one step. The page describes only **one** of them, which is why it says so little. `junocam2isis` **imports a raw JunoCam image and splits it into separate files per filter and per framelet**, methane included. The usual chain (standard ISIS workflow, not yet tested by us) is:

1. `junocam2isis`: import and split by filter;
2. `spiceinit`: attach Juno's position and pointing;
3. `cam2map`: project each strip onto a latitude/longitude map;
4. `automos`: stitch the strips into one image per filter.

The result is red, green, blue and **methane images aligned on the same map**, which are exactly the pairs we need.
- **Good:** free, official, handles all 4 filters, and used in published research.
- **Bad:** it runs on Linux/Mac, so on Windows we'd need **WSL**. It also needs a large SPICE download, and the learning curve is steep.

### Bonus: `junocam_projection`
`https://github.com/ramanakumars/JunoCamProjection`

This Python package comes from **Ramanakumar Sankar**, the same person behind Jovian Vortex Hunter, so it probably produced our Hugging Face crops. We checked the code: it **only supports red, green and blue**. The filters are hard-coded as B, G and R, it assumes 3 strips per frame, and its brightness correction covers only those 3 strips. It also only runs on Linux. Adding methane would mean modifying someone else's code, which could cost us a week or more. **It's useful for more RGB crops, not for methane.**

**Tool verdict: ISIS is the only option that handles methane without changing anyone's code.**

## 5. What the data actually contains

Using the `INDEX.TAB` files of all 35 volumes, we counted every JunoCam image: about **42,000 raw images, orbits 0–80**.

**Good news:** methane images exist in **almost every orbit** (all except 71 and 76). Orbits 13–36 alone have **755 methane images**.

**Three catches:**
1. **Methane is always a separate photo.** Only 10 images in the whole mission use all four filters at once (letter A), and **none of them show Jupiter**. Every methane image of Jupiter is a methane-only photo (M), taken **about 30 seconds** away from the nearest RGB photo (20–150 s for the close-ups below). The clouds don't change in that time, but Juno moves, so the two photos see Jupiter from different angles. **To make pixel-aligned pairs we must project both onto a map** (the ISIS step above). There's no shortcut. Zsolt is right that it's the same sensor, but in practice they're two files.
2. **Most are taken from far away.** 429 of the 755 are a "departure movie" shot as Juno leaves, at a median distance of about 316,000 km. That's too blurry for small features.
3. **The close-ups are mostly polar.** Only **101 unique close-up methane images** (closer than 60,000 km) exist in orbits 13–36. They mostly show the **north pole and its circumpolar cyclones** (about 51), plus jets (13), folded filamentary regions (7), hot spots (7), test images (12), the Great Red Spot (4) and a few belts (5). **These are mostly not the mid-latitude regions our vortex labels cover**, so combining methane with our labels would only work for a small subset.

### The ready-made methane maps (letter H)

Inside `DATA/GLOBAL_MAPS/`, the JunoCam team already **projected a few photos per orbit onto a flat map of the whole planet**: one methane map (H) and one RGB map (P). They exist for **71 orbits**, including all of 13–36, and the methane and RGB maps of an orbit are on **the same grid**. So these would be ready-made aligned pairs, without ISIS. We checked orbit 13 in detail:
- **coarse:** about **78 km per pixel**, against about 18 km per pixel in our Hugging Face crops. A typical vortex would be only about 16 pixels wide;
- **small coverage:** each map is built from just 2 photos, and both maps together cover only **~26%** of the planet;
- **very few brightness levels:** 8-bit and uncalibrated, with useful methane values spanning only about **20 grey levels**;
- **noisy:** seams between photos and saturated white patches. The storms visible in RGB couldn't be seen in the methane map.

**Verdict: not good enough for vortex-scale work or for training a model.** We checked only orbit 13, but the other maps are built the same way.

### The close-ups, up close

We downloaded 4 calibrated methane close-ups with their RGB partners and looked at them side by side:

| Image | Verdict | Why |
|---|---|---|
| **PJ27, northern circumpolar cyclones** | ✅ good | swirls, filaments and vortices clearly visible in methane |
| **PJ27, Jet N5** | ✅ mostly good | lots of cloud detail and an oval vortex visible, but some bright clouds are overexposed |
| **PJ21, Great Red Spot** | ⚠️ half unusable | about half of the strips are saturated (pure white): the Great Red Spot reaches so high that it's very bright in methane, and the long exposure overflowed |
| **PJ13, Hot Spot** | ❌ unusable | flat, blocky and banded, from early orbits when settings were still being tested |

**General observations:**
- Methane exposures are **100–200 ms against 3.2 ms for RGB**, which is why bright clouds saturate.
- Some methane images use **lossy compression**.
- Quality seems to **improve in later orbits**: the good examples are from PJ27, and orbits 23–27 contain "methane test with long exposure" entries. This is based on only 4 images.

## 6. What it all means for us

**What's good:**
- The idea is **new** (we found no paper combining RGB with JunoCam methane), and the methane image is physically meaningful: it's a cloud-height map.
- The data is **free, public and well documented**, and every methane close-up has an RGB partner taken seconds apart.
- In good images, the methane band shows **real structure**, including vortices.

**What's bad:**
- **No ready-made pairs.** The global maps are too coarse, so we must **project the images ourselves with ISIS**. That's the biggest piece of work, especially on Windows.
- The good close-ups in orbits 13–36 are **few (a few dozen usable pairs at most) and mostly polar**, so they barely overlap with our **mid-latitude vortex labels**.
- Some images are **saturated** or **compressed**, so we'll have to filter them.

**Our options:**
1. **Darius's idea on its own (predict methane, i.e. cloud height, from RGB).** It doesn't need our labels and can use **any orbit**. Later orbits have many more methane images, so adding them would grow the dataset. Start with a small U-Net, then try the two-encoder model, as Zsolt advised.
2. **Combined with vortices ("are some vortices higher?").** It only works where methane close-ups and our labels overlap, which is likely a small set. The polar cyclones and the Great Red Spot have methane images but are probably not in our labels.
3. **Stay with the original JOVIS plan** (vortex detection on the Hugging Face crops), with the methane band as a later extension.

**Suggested next steps if we go with methane:**
1. Automatically check all 101 close-ups (and later orbits): saturation percentage and compression type, to get the exact number of usable pairs. That's about 3 GB to download.
2. Install ISIS (via WSL) and **project one good pair end to end**, for example PJ27 Jet N5, to see how long it takes.
3. Cut aligned pairs into patches and train a **small U-Net baseline** (RGB → methane).

### Is there enough data for machine learning?

The 101 close-ups above are **only from orbits 13–36**. Across the whole mission there are **312 close-up methane images**: 35 in orbits 0–12, 101 in 13–36 and 176 in 37–80. There are about 10,200 methane images in total, but most are taken from far away.
- **Enough for a small model:** each projected image can be cut into hundreds of patches, so even if only about half are usable (~150 pairs), we'd have tens of thousands of patches. That's fine for a small U-Net.
- **Not enough for a big model** trained from scratch.
- **Split by orbit, never by patch:** patches from the same image are nearly identical, so a random split would make the results look better than they really are.
- **Not yet verified:** how many of the 312 are actually usable. So far we've looked at only 4, and 2 were good.

---

### Files produced during the investigation

These are on Rareș's computer (`Rospin/teste/`), not in this repo yet:
- `junocam_image_index.csv`: every JunoCam image of the mission (orbit, filter letter, time, description, distance);
- `methane_closeups/closeup_methane_13_36.csv`: the 101 close-up methane images in orbits 13–36, with their RGB partner and download paths;
- `methane_closeups/rgb_vs_methane_*.png`: the four side-by-side comparisons from the table above;
- `pj13_methane_vs_rgb.png`, `pj13_rgb_vs_methane_zoom.png`: the orbit 13 global maps.
