---
title: "Dealing with PDFs in the command line"
date: 2022-05-22T20:29:37-03:00
draft: false
---

# Generating a PDF from images

You can use [`img2pdf`](https://gitlab.mister-muffin.de/josch/img2pdf):

```sh
img2pdf -o out.pdf image1.png image2.png
```

Replace `out.pdf` with a output filename and `image1.png image2.png` with the images you want joined in the PDF (feel free to use `*`).

If some of your images are low-resolution and others are high-res, it helps to give the program a standard page size (`-S`) for all images; otherwise, each page will have a very different size. It accepts international standard paper sizes (see the `--help` for which) as well as `WIDTHxHEIGHT` with regular units (`cm`, `mm`, `in`...):

```sh
img2pdf -S200cmx200cm -o out.pdf smallimage.png bigimage.jpg
```

ImageMagick's `convert` utility is more widespread and works with more formats, but causes noticeable quality loss during conversion, so I don't recommend it.

# Converting office documents to PDF

LibreOffice provides a `--convert-to` option that works without spawning a GUI. It creates a file of the same name as the original but with changed extension on the shell's working directory.

Say you run the following on a terminal:

```sh
$ cd ~/Documents
$ soffice --convert-to pdf ~/Downloads/assignment.docx
```

This will create/**overwrite**(!) `assignment.pdf` on your `Documents` folder (since you `cd`'d there).

`--convert-to` accepts every extension that LibreOffice can read/write; you could use it for headless docx → odt, ppt → odp, etc. conversions.

# Joining PDFs

Poppler comes with `pdfunite`, which can be used like this:

```sh
pdfunite pdf1.pdf pdf2.pdf out.pdf
```

- Never forget the `out.pdf` when using globs.

# Extracting page ranges from a PDF

[`pdftk`](https://gitlab.com/pdftk-java/pdftk) is really a Swiss Army Knife for dealing with PDFs with many subcommands. But, focusing on the basics, you can use `pdftk in.pdf cat STARTPAGE-ENDPAGE output out.pdf` for extracting page ranges within a PDF:

```sh
pdftk assignment.pdf cat 1-5 ouput firstpagesofassignment.pdf
# Instead of number, you use the special keyword `end` so it prints everything towards the end
pdftk assignment.pdf cat 400-end ouput lastpagesofassignment.pdf
```

- Another possibility is writing a rather complicated script with Poppler's `pdfseparate` (which can only extract individual pages to individual PDFs) and `pdfunite`, but `pdftk` is much more convenient.

# Removing annotations

Another use for `pdftk` is removing annotations (virtual highlights, text, etc.) from PDFs.

This is adapted from [this GitHub Gist and its comments](https://gist.github.com/stefanschmidt/5248592).

```
#!/bin/sh
for file; do
	pdftk "$f" output - uncompress |
		LANG=C LC_CTYPE=C sed '\|^/Annots|d;\|/Type /Annot|d' |
		pdftk - output "${f%.pdf}.unannot.pdf" compress
done
```

# Compressing PDFs

[Alfred Klomp's `shrinkpdf`](https://github.com/aklomp/shrinkpdf) is my choice for compressing PDFs with high-resolution images. I've made [a variant](https://gist.github.com/capezotte/1bf25c00d9abab0d622d48d37c860316) that mixes `gs` and `pdftk` to try to conserve text resolution better.
