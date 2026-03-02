---
title: "Why SuperDoc Shows Only the Last Image: A DOCX Internals Investigation"
date: 2026-03-01 19:30:00 +0530
categories: [Engineering, Debugging]
tags: [docx, superdoc, ooxml, proseMirror, debugging]
---

# Why SuperDoc Shows Only the Last Image: A DOCX Internals Investigation

While integrating DOCX generation into our document editor pipeline, I ran into one of the most confusing rendering issues I have seen so far.

A DOCX generated with multiple images displayed perfectly in Word, LibreOffice, and Google Docs. But inside the SuperDoc editor, every image slot showed the same image. Always the last one.

This post is not about a final fix yet. It is a breakdown of the debugging journey and what I discovered while tracing the root cause.

---

## The Symptom

The generated report contained multiple property images arranged in a grid.

Expected inside editor:

* Image 1 → slot 1
* Image 2 → slot 2
* Image 3 → slot 3

Actual result inside SuperDoc:

* Slot 1 → last image
* Slot 2 → last image
* Slot 3 → last image

Every slot displayed the same image.

Strangely:

* Microsoft Word → correct
* LibreOffice → correct
* Google Docs → correct
* SuperDoc editor → wrong

So the DOCX itself was valid. The issue appeared editor-specific.

---

## Environment

| Tool           | Version |
| -------------- | ------- |
| SuperDoc       | 1.16.x  |
| docx-templates | 4.15.0  |
| Node           | 20.x    |

Images inserted using `docx-templates` loops and loaded into editor via `docxUrl`.

---

## First Check: Is DOCX Corrupt?

Initial assumption was that the DOCX generation might be broken.

So I verified:

* Opened in Word
* Opened in LibreOffice
* Converted to PDF
* Uploaded to Google Docs

Everything rendered correctly everywhere except inside the editor.

That ruled out basic DOCX corruption.

---

## Deep Debugging Approach

Since DOCX files are just zip archives, I extracted and inspected raw XML:

```

word/document.xml
word/_rels/document.xml.rels
word/media/

```

Goal was to see how images were referenced internally.

---

## Finding 1: Non-Standard Relationship IDs

Normally DOCX uses relationship IDs like:

```

rId1
rId2
rId3

```

But generated DOCX contained:

```

img2073076884
img99887766

````

Custom hash-based IDs.

Example:

```xml
<Relationship Id="img2073076884"
 Type=".../image"
 Target="media/template_document.xml_img2073076884.jpg"/>
````

This is valid but not standard.

### Test performed

Converted all relationship IDs to standard `rId1`, `rId2`, etc and updated references.

Result:
No change in editor behaviour.
All image slots still showed the last image.

So relationships were not the main cause.

---

## Finding 2: Media Filenames

Generated filenames looked like:

```
template_document.xml_img2073076884.jpg
```

Instead of standard:

```
image1.jpg
image2.jpg
```

Renamed them to standard format and updated references.

Result:
Still broken in editor.

So filename convention was not the cause.

---

## Finding 3: Images Inside Tables?

Another suspicion was that images inside table cells were being flattened or merged.

Checked raw XML structure.

Images were already in standalone paragraphs.
Not nested inside complex table drawing structures.

Result:
No structural issue found.

---

## Finding 4: Base64 External References

As a further isolation step, I also converted images to base64-encoded external references to rule out any media file resolution issue inside the editor.

Result:
Still broken. All slots continued showing the last image.

At this point, the problem had to be in how the editor was interpreting drawing node identity, not how images were stored or referenced.

---

## The Strongest Lead So Far (Unverified)

While inspecting drawing XML for each image, one pattern appeared:

```xml
<pic:cNvPr id="0" name="image1.jpg"/>
<pic:cNvPr id="0" name="image2.jpg"/>
<pic:cNvPr id="0" name="image3.jpg"/>
```

Every image had:

```
id="0"
```

According to the OOXML spec, this attribute is meant to be a unique identifier per drawing object within a document.

But `docx-templates` appears to be assigning `id="0"` to all generated images.

My working theory: SuperDoc uses ProseMirror under the hood for document rendering. If ProseMirror or SuperDoc's DOCX parser is using `pic:cNvPr id` to distinguish drawing nodes, duplicate IDs across all images could cause it to treat them as the same node, with the last image loaded winning every slot.

That said, this is a hypothesis, not a confirmed fix. The next step is to patch `docx-templates` output to inject unique sequential IDs, do a clean server restart, and verify the result properly. A failed test earlier in this session turned out to be against old compiled code because the server hadn't been restarted after a rebuild.

---

## What Has Been Tried So Far

| Attempt                                | Result                                     |
| -------------------------------------- | ------------------------------------------ |
| Extract images from table              | No change                                  |
| Rename relationship IDs to rId1/rId2   | No change                                  |
| Rename media filenames to image1.jpg   | No change                                  |
| Convert images to base64 external refs | No change                                  |
| Inspect drawing XML                    | Found duplicate `id="0"` across all images |
| Patch to unique IDs                    | Not fully verified yet                     |

---

## Current Status

Issue is still under investigation.

Leading hypothesis: duplicate `pic:cNvPr id="0"` across all images is causing the editor to collapse them into a single node.

Next step: inject unique IDs, do a full server restart, and run a clean verification.

---

## Lessons From This Debugging Session

### 1. If it works in Word, it can still be wrong

Word tolerates many OOXML violations silently. Editors and converters are stricter. Never assume Word rendering equals correctness.

### 2. Always inspect DOCX internally

DOCX debugging becomes much faster once you treat it as a zip file. Unzip it, inspect the XML, search for patterns, verify structure.

### 3. Rendering engines behave differently

| Tool           | Strictness     |
| -------------- | -------------- |
| Word           | Very forgiving |
| Google Docs    | Moderate       |
| LibreOffice    | Strict         |
| Editor engines | Very strict    |

Editor-specific bugs often come from structural assumptions the spec makes but Word quietly ignores.

---

## Why I'm Documenting This

This bug is still unresolved. But the debugging journey itself surfaced important learnings about DOCX internals, rendering engine strictness, and real-world document pipelines.

If you've encountered similar behaviour with DOCX image rendering, I'd genuinely like to hear what you found.
