# About the figures


## Old figures: omnigraffle

Old figures are made with omnigraffle and exported to pdf and png.
Latex output uses pdf.

The problem with these is that keeping the same look and feel over all of these is very complicated.

## New figures: asciiart -> svg -> pdf

Using asciiart allows me (Guille) to keep the same look and feel in the figures.

### Editing your figures online

The canonical editor is:

https://ivanceras.github.io/svgbob-editor/

You can help yourself using https://asciiflow.com/#/ but it is not guaranteed that you will have the same output

### Converting them to pdf

Converting ascii art to pdf requires two command line tools: `aasvg` and `rsvg-convert` which I installed as follows in Mac:

```bash
npm install -g aasvg
sudo apt-get install librsvg2-bin
```
If the previous apt-get does not work

Notice that you should also install svgbob

```
brew install librsvg
brew install svgbob
```

Figure source code is asciiart and named `*-ascii.txt`.
We then convert them to svg using the following commands.

**TIP: Use the makefile ;)**

```
aasvg < x.txt > x.svg
rsvg-convert -f pdf -o x.pdf x.svg
```