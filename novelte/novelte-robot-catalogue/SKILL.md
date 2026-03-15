---
name: novelte-robot-catalogue
description: >
  Creates a Novelte-branded product catalogue in PPTX format for a new robot launch.
  Trigger when the user provides factory materials (PDF or PPTX) and asks to produce
  a Novelte catalogue, product deck, or robot brochure. Handles both PDF and PPTX input.
  Always uses the Novelte Product Catalogue Template as the base file.
  Includes known pitfalls from real sessions: clean.py ordering, floating text box rendering,
  and pack.py validation bypass.
license: Proprietary — Novelte Robotics internal use only
---

# Novelte Robot Catalogue Skill

This skill turns raw factory materials into a finished Novelte-branded product catalogue,
following the standardised structure used across all Novelte robot launches.

---

## Prerequisites

```bash
pip install "markitdown[pptx]" --break-system-packages
pip install Pillow --break-system-packages
```

The base template must exist at:
`/mnt/user-data/uploads/Novelte_Product_Catalogue_Template.pptx`

---

## Workflow Overview

```
1. INGEST       — Extract all content from factory materials
2. ASSESS       — Decide slide structure based on content richness
3. BUILD        — Produce the PPTX from template
4. QA           — Visual inspection and fix loop
5. DELIVER      — Output to /mnt/user-data/outputs/
```

---

## Step 1 — Ingest Factory Materials

### If input is PPTX:
```bash
python -m markitdown factory_file.pptx
```

### If input is PDF:
```bash
python -m markitdown factory_file.pdf
```

Extract and organise the following from the source material:

| Field | Notes |
|-------|-------|
| **Robot name** | Exact product name as it should appear on the cover |
| **Tagline / subtitle** | One-line description of the robot's category or key benefit |
| **Hero product image** | Best available image of the robot (extract from source or flag for user to supply) |
| **Capabilities list** | List of key capabilities — titles + short descriptions + icons if available |
| **Feature details** | For each feature: title, description, supporting images |
| **Specification table** | All spec rows: parameter name + value |
| **Spec product image** | Side/front view of the robot for the spec slide |
| **Additional images** | Any scenario, component, or lifestyle images |

### Image Extraction
```bash
# Unpack PPTX to access embedded images
python /mnt/skills/public/pptx/scripts/office/unpack.py factory_file.pptx factory_unpacked/
ls factory_unpacked/ppt/media/
```

For PDFs, note page numbers containing images and flag them for the user:
> "I found images on pages X, Y, Z. Please confirm which ones to use, or supply replacements."

---

## Step 2 — Assess Content & Plan Slide Structure

After ingesting, evaluate the detail level of each feature to decide the middle slides structure.

### Decision Rule for Middle Slides

```
FOR EACH feature or topic in the factory material:
  IF the factory material provides:
    - A clear feature title
    - At least 2–3 sentences of description OR a supporting image
    → Give it its OWN slide (use feature name as heading)
  
  ELSE (brief mention, single sentence, no image):
    → COMBINE 2–3 such features into one "Highlights" slide
```

### Output: Slide Plan

Produce a numbered slide plan before building. Example:

```
Slide 1  — Cover          [Robot Name + Tagline + Hero image]
Slide 2  — Capabilities   [Overview grid of all key capabilities]
Slide 3  — [Feature A]    [Own slide — rich detail available]
Slide 4  — [Feature B]    [Own slide — rich detail available]
Slide 5  — Highlights     [Combined: Feature C + D + E — brief info only]
Slide 6  — Highlights     [Combined: Feature F + G — brief info only]
Slide 7  — Specification  [Spec table + robot image]
Slide 8  — Closing        [Fixed — no changes needed]
```

**Show this plan to the user and get confirmation before building.**

---

## Step 3 — Build the PPTX

### 3a. Unpack the Template

```bash
cp /mnt/user-data/uploads/Novelte_Product_Catalogue_Template.pptx /home/claude/working_template.pptx
python /mnt/skills/public/pptx/scripts/office/unpack.py /home/claude/working_template.pptx /home/claude/unpacked/
```

### 3b. Understand the Template Slide Map

The template contains these slides (in order):

| Slide file | Purpose | Action |
|------------|---------|--------|
| `slide1.xml` | Cover | KEEP — edit robot name, tagline, swap hero image |
| `slide2.xml` | Capabilities (empty 2×3 grid) | KEEP — populate with capability cards |
| `slide3.xml` | Highlights (2-column, 2 features) | KEEP as first middle slide — edit or duplicate |
| `slide4.xml` | Highlights duplicate | KEEP or DELETE depending on slide plan |
| `slide5.xml` | Highlights duplicate | KEEP or DELETE depending on slide plan |
| `slide6.xml` | Specification | KEEP — edit spec table rows + swap robot image |
| `slide7.xml` | Closing / Thank You | KEEP — never edit this slide |

### 3c. Structural Changes (do ALL before editing content)

**Adding slides** (for own-feature slides or extra Highlights):
```bash
# Duplicate a Highlights slide as the base for a new detail slide
python /mnt/skills/public/pptx/scripts/add_slide.py /home/claude/unpacked/ slide3.xml
# Copy the printed <p:sldId> line into ppt/presentation.xml at the correct position
```

**Deleting slides** (remove unneeded Highlights duplicates):
- Remove the `<p:sldId>` entry from `ppt/presentation.xml`
- Run clean afterwards

**Ordering**: Ensure `ppt/presentation.xml` `<p:sldIdLst>` matches the slide plan exactly.

### 3d. Edit Slide Content

#### SLIDE 1 — Cover
Fixed elements (DO NOT change):
- Dark split diagonal background (the background shape/image is part of the template)
- Novelte logo position (top-right)
- Decorative orange/red diagonal lines

Variable elements to update in `slide1.xml`:
- **Robot Name**: Large white text (the first `<a:t>` in the title text box)
  - Format: `<robot_name>` in white (`#FFFFFF`), Roboto Black, ~54pt
  - The last word or key product identifier is styled in Novelte Orange (`#FCA311`)
- **Tagline**: Smaller white/light text below the name, Roboto, ~18pt
- **Hero image**: Replace the robot image (`<p:pic>` element) with the new robot's image

#### SLIDE 2 — Capabilities
Layout: 2×3 grid of rounded-corner image cards (or icon+text layout depending on content).

For each of the 6 capability slots:
- Replace the image/icon in each card
- Add capability title (bold, `#FCA311` orange, Roboto Bold ~14pt)
- Add short description (Roboto, `#3A3838`, ~11pt)

If the robot has more or fewer than 6 capabilities:
- Fewer than 6: Remove excess card elements entirely (don't just blank them)
- More than 6: Use a different layout — icon+text rows (like S55 Pro) to fit more items

#### SLIDES 3–N — Middle / Detail Slides

**For "Own Feature" slides** (rich detail, feature name as heading):
- Update slide title to the feature name (orange, Roboto Black)
- Left column: feature description text (2–4 paragraphs or bullet points)
- Right column: supporting image(s)
- If multiple sub-features exist on one slide, use orange subheadings for each

**For "Highlights" slides** (combined brief features):
- Keep heading as "Highlights"
- Two-column layout: left feature + right feature
- Each feature: orange title (Roboto Bold ~16pt) + 2–3 sentence description + image below
- Maximum 3 features per Highlights slide; create a new slide if more are needed

**Heading style for all content slides** (matches template):
```xml
<!-- Orange title, Roboto Black -->
<a:r>
  <a:rPr lang="en-US" sz="3600" b="1" dirty="0">
    <a:solidFill><a:srgbClr val="FCA311"/></a:solidFill>
  </a:rPr>
  <a:t>Feature Title Here</a:t>
</a:r>
```

#### SLIDE — Specification
Layout: Orange-labelled table on left + robot product image on right.

**Hard limit: maximum 10 spec rows.** Do not copy every spec from the factory sheet — curate the most customer-relevant ones.

##### Mandatory specs (always include for every robot):
| Spec | Notes |
|------|-------|
| Dimension | L×W×H in mm |
| Weight | in kg |
| Operation Time | runtime per charge |
| Charging Time | time to full charge |

##### Robot-type specs (fill remaining rows up to the 10-row cap):

**Do not assume the robot type — identify it from the factory materials each time.**
Read the product description and feature content to determine what category the robot belongs to (e.g. floor cleaning, pool cleaning, delivery, disinfection, reception, security patrol, lawn mowing, etc.). There is no fixed list of robot types — use judgement.

Once the type is identified, select the specs from the factory sheet that are most meaningful for that category. Examples by type:

| Robot Type | Relevant additional specs to consider |
|------------|---------------------------------------|
| Floor cleaning | Cleaning Efficiency, Cleaning Width, Water Tank Capacity, Dust Bag Capacity, Noise Level, Cleaning Modes |
| Pool cleaning | Pool Coverage Area, Filter Mesh Size, Suction Power, Cleaning Modes, Waterline Cleaning, Cable Length |
| Delivery / service | Max Load Capacity, Number of Cabinets, Cabinet Size/Volume, Screen Size, Speed, Network |
| Disinfection | Disinfection Area, UV/Spray Coverage, Tank Capacity, Disinfection Modes |
| Reception / patrol | Speed, Navigation Type, Screen Size, Network, Obstacle Avoidance Sensors |
| Other types | Use the same principle — pick specs that communicate unique value to a buyer |

This table is illustrative, not exhaustive. Always derive the right specs from the actual robot.

##### Selection rule:
> From the factory spec sheet, pick the additional rows that best communicate the robot's unique value to a buyer — up to a total of 10 rows including the 4 mandatory ones. Skip specs that are too technical, redundant with the feature slides, or not meaningful to a non-engineer audience. Min. Passable Path Width and Slope Climbing Angle are optional — include them only if they are a meaningful selling point for that robot (e.g. a robot designed for narrow corridors or sloped environments).

- Table rows: orange label cell (left) + value cell (right)
- Replace robot image with spec/front-view product photo

#### LAST SLIDE — Closing (NEVER EDIT)
This slide is 100% fixed. Do not modify any element on it.

### 3e. Known Pitfalls — Read Before Clean and Pack

**1. Edit `presentation.xml` AFTER `clean.py`, never before**

`clean.py` silently restores `presentation.xml` from the original template, wiping any slide deletions or reordering you've done. The correct order is:

1. Use `add_slide.py` to create any new slides (registers them in rels / Content_Types)
2. Run `clean.py` — let it clean against the original structure
3. **Then** edit `ppt/presentation.xml` `<p:sldIdLst>` to set the final order/deletions
4. Edit slide content
5. Pack

If you edited `presentation.xml` first and `clean.py` reset it, just re-apply your `<p:sldIdLst>` changes after the clean.

---

**2. Text must go inside the shape — never in a separate floating text box**

If you create a background shape (card, pill, placeholder rectangle) and then add a separate `txBox` floating on top of it, LibreOffice will silently drop the text box. The shape renders; the text disappears.

Always put text directly inside the shape's own `<p:txBody>`. One `<p:sp>` = shape fill + text together.

```xml
<!-- ✅ CORRECT: text inside the shape -->
<p:sp>
  <p:nvSpPr><p:cNvPr id="10" name="card1"/><p:cNvSpPr/><p:nvPr/></p:nvSpPr>
  <p:spPr>
    <a:xfrm>...</a:xfrm>
    <a:prstGeom prst="roundRect">...</a:prstGeom>
    <a:solidFill>...</a:solidFill>
  </p:spPr>
  <p:txBody>
    <a:bodyPr anchor="ctr" lIns="91425" tIns="91425" rIns="91425" bIns="91425"><a:normAutofit/></a:bodyPr>
    <a:lstStyle/>
    <a:p><a:r><a:rPr lang="en-US" sz="1300" b="1">...</a:rPr><a:t>Title</a:t></a:r></a:p>
    <a:p><a:r><a:rPr lang="en-US" sz="1000">...</a:rPr><a:t>Description text.</a:t></a:r></a:p>
  </p:txBody>
</p:sp>
```

---

**3. If `pack.py` throws an ID uniqueness error, use `--validate false`**

The validator occasionally produces false positives due to a Cython bug:

```
FAILED - Found 2 ID uniqueness violations:
  ppt/slides/slideN.xml: Error: argument of type 'cython_function_or_method' is not iterable
```

IDs only need to be unique *within* a single slide, not across all slides. If you've verified your per-slide IDs are unique, bypass with:

```bash
python scripts/office/pack.py unpacked/ output.pptx --original template.pptx --validate false
```

The file will open and render correctly in both PowerPoint and LibreOffice.

---

### 3f. Clean and Pack

```bash
python /mnt/skills/public/pptx/scripts/clean.py /home/claude/unpacked/
python /mnt/skills/public/pptx/scripts/office/pack.py /home/claude/unpacked/ /home/claude/output_catalogue.pptx --original /home/claude/working_template.pptx
```

---

## Step 4 — QA

### Text QA
```bash
python -m markitdown /home/claude/output_catalogue.pptx
```
Check: correct robot name, no placeholder text left, all specs present, closing slide unchanged.

```bash
python -m markitdown /home/claude/output_catalogue.pptx | grep -iE "\bRobot Name\b|\bTaglines\b|\bHighlights\b.*Details|\bXXX\b|lorem|ipsum|\[insert"
```
Any match = unfilled placeholder. Fix before proceeding.

### Visual QA
```bash
python /mnt/skills/public/pptx/scripts/office/soffice.py --headless --convert-to pdf /home/claude/output_catalogue.pptx
rm -f /home/claude/slide-out-*.jpg
pdftoppm -jpeg -r 150 /home/claude/output_catalogue.pdf /home/claude/slide-out
ls -1 "$PWD"/slide-out-*.jpg
```

Inspect each slide image. Look for:
- [ ] Cover: correct robot name + tagline, hero image visible, background unchanged
- [ ] Capabilities: all cards populated, no empty cards remaining
- [ ] Middle slides: correct headings, text fits within boxes, images not stretched
- [ ] Spec slide: all rows present, correct values, robot image correct
- [ ] Closing slide: identical to template — addresses, phone, email, website all intact
- [ ] No overlapping text or cut-off content
- [ ] No leftover placeholder text anywhere

Fix all issues, repack, re-render, and re-inspect until a full clean pass.

---

## Step 5 — Deliver

```bash
cp /home/claude/output_catalogue.pptx "/mnt/user-data/outputs/Novelte_[RobotName]_Product_Catalogue.pptx"
```

Name format: `Novelte_[RobotName]_Product_Catalogue.pptx`
Examples:
- `Novelte_S55_Pro_Product_Catalogue.pptx`
- `Novelte_Wybot_M2_Product_Catalogue.pptx`

Then use `present_files` to share with the user.

---

## Brand Constants — Quick Reference

| Element | Value |
|---------|-------|
| Primary accent | `#FCA311` (Novelte Orange) |
| Body / heading text | `#3A3838` (Charcoal) |
| Background | `#FFFFFF` (White) |
| Dark panels | `#000000` / `#1A1A1A` |
| Heading font | Roboto Black |
| Body font | Roboto |
| Slide size | 16:9, 13.33" × 7.50" |
| Logo: dark bg | white version |
| Logo: light bg | dark version |

---

## Handling Missing Images

**No slide should ever be text-only.** Every slide must have at least one visual element — photo, illustration, or icon. This is a hard rule.

For every slide that lacks a supplied image, work through this priority order:

1. **Search the web for a real photo** — search for the most relevant image:
   - Hero shots: `[robot name] [manufacturer]`
   - Environment/lifestyle: relevant scenarios (e.g. "office security patrol robot", "hotel service robot lobby")
   - Feature illustrations: relevant technology or concept visuals

2. **If no suitable photo found, search for a cartoon icon or illustration** — a clean, flat-style icon is always better than no visual. Search for:
   - `[feature name] flat icon illustration`
   - `[concept] cartoon vector icon`
   - Prefer simple, clean styles that won't clash with the Novelte brand

3. **Present all candidate visuals to the user** — show which image/icon is intended for which slide and ask for approval before embedding:
   > "Here are the visuals I found for the missing slots — please confirm which to use before I build."

4. **Only embed approved visuals** — never embed web-sourced images or icons without explicit user confirmation.

5. **Last resort only** — if the user explicitly declines all options and provides nothing, insert a placeholder:
   - Grey rectangle with orange dashed border
   - Text: `[IMAGE NEEDED: description]`

---

## Notes on Content Tone

When rewriting factory content into Novelte's voice:
- **Be concise**: One idea per sentence. Remove filler words.
- **Lead with benefit**: Start descriptions with what the robot *does for the user*, not technical specs.
- **Avoid factory-style translation issues**: Fix grammar, unnatural phrasing, and overly technical Chinese-translated English.
- **Keep specs verbatim**: Do not paraphrase numbers, dimensions, or technical measurements.
- **Capability titles**: Short, punchy — 2–4 words max (e.g. "Ultra-Quiet Operation", "AI Vision Mode", "Auto Recharging").
