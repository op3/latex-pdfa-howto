# Create PDF/A compliant document from LaTeX

pdflatex is itself capable of creating [PDF/A](https://en.wikipedia.org/wiki/PDF/A) compliant documents using hyperref.
While it is always possible for the document to *claim* compliance, in reality, using different fonts and including images can easily result in violations of the standard that are not easily fixable.
In the following, I will first outline, how one can create a PDF that is as compliant as possible using pdflatex.
Secondly, the resulting PDF file is fixed to restore PDF/A compatibility.

For demonstration purposes, a PDF1.7 file compliant with PDF/A-2b will be created, using an sRGB ICC profile.
Using lualatex or xelatex instead is also possible with some modifications.

## Create PDF/A document using pdflatex

A few modifications are necessary to the `.tex`-file itself.
At the very top (even before `\documentclass`), the following lines are necessary:

```
%\pdfobjcompresslevel=0
\pdfminorversion=7
\pdfinclusioncopyfonts=1
```

`\pdfobjcompresslevel=0` is used to disable the compression of object streams and only required for PDF/A-1b, so we can skip it.
`\pdfminorversion=1` forces the usage of PDF1.7.
`\pdfinclusioncopyfonts=1` forces the inclusion of all necessary fonts.

Following the `\documentclass` declaration, the following code should be added to the preamble:

```
\input{glyphtounicode.tex}
\input{glyphtounicode-cmr.tex}
\pdfgentounicode=1
\usepackage{cmap}
```

Both `.tex`-files should be provided by your TeX-distribution.
They ensure the correct mapping of (most?) characters used by LaTeX to appropriate Unicode characters.
`cmap` ensures that the resulting PDF file is searchable and copyable.

Finally, [`hyperref`](https://ctan.org/pkg/hyperref) is used to control various PDF features (including PDF/A compliance).
`hyperxmp` adds support for XMP metadata.
For a complete overview over all supported metadata fields, refer to the [hyperxmp documentation](https://ctan.org/pkg/hyperxmp).
The following code has to be added at the very end of your preamble (Only `cleveref` should be included even later):

```
\usepackage{hyperxmp}
\usepackage[pdfa]{hyperref}
\hypersetup{%
    unicode,
    pdfapart=2,
    pdfaconformance=B,
    pdfauthor={Author Name},
    pdfdate={2018-07-13},
    pdftitle={Title of the work},
}

\input{pdfacode}
```

The `pdfacode.tex` file is available [in this repository](pdfacode.tex).
It forces the inclusion of the ICC profile.
An instance of the used profile `sRGB.icc` can be downloaded from [from CTAN](https://ctan.org/pkg/colorprofiles).
It is probably best to copy this file to your project folder.

The created PDF file should either be compliant with PDF/A or pretty close.
[veraPDF](http://verapdf.org/) can be used as a validator.

If you want to use xelatex or lualatex instead, create a PDF/X document or use a CMYK ICC profile and color space, [this answer on StackExchange](https://tex.stackexchange.com/a/349521) should be helpful.


## Create PDF/A document using Ghostscript + exiftool + qpdf

If necessary, [Ghostscript](https://www.ghostscript.com/) can be used to finally create a valid PDF/A document.
Please note that it is not sufficient to use Ghostscript without the modifications above.
Use the following command to convert your PDF file:

```
gs -P -dPDFA=2 -dNOOUTERSAVE -sProcessColorModel=DeviceRGB -sColorConversionStrategy=RGB -dCompatibilityLevel=1.7 -sDEVICE=pdfwrite -o intermediate.pdf -dPDFACompatibilityPolicy=1 PDFA_def.ps mydocument.pdf
```

For this, the file [PDFA\_def.ps](PDFA_def.ps) is required.
It also relies on the ICC profile used above.
Modifications are necessary if a different ICC profile is being used.
The advantage of using Ghostscript is, that the resulting PDF file will be more optimized and probably has a smaller file size.
This command will automatically convert all included colors to the target color profile.

While Ghostscript is very capable of manipulating and optimizing the PDF itself, it does not care too much about metadata, resulting in several lost or modified metadata entries.
To fix this, [exiftool](https://www.sno.phy.queensu.ca/~phil/exiftool/) is used to transfer the metadata of the original document created by pdflatex to the PDF file created by Ghostscript.
As of writing, exiftool does not yet support all XML tags and namespaces required for PDF/A compliance.
The file [pdfa.config](pdfa.config) provides exiftool with the necessary information to support writing (and copying) these tags:

```
exiftool -config pdfa.config -TagsFromFile mydocument.pdf "-all:all>all:all" "-xmp-dc:all>xmp-dc:all" "-pdf:subject>pdf:subject" -overwrite_original intermediate.pdf
```

It is also possible to add additional metadata in this step.
The modifications conducted by exiftool are reversible, meaning that the original metadata created by Ghostscript are still contained in the file.
Some PDF readers (and PDF/A validators) will complain about this, so a further step is necessary.
[qpdf](https://github.com/qpdf/qpdf) is used to linearize the PDF file (while maintaining PDF/A compliance):

```
qpdf --linearize --newline-before-endstream intermediate.pdf final.pdf
```

That’s it!
You have successfully created a fully PDF/A-2b compliant PDF file.


