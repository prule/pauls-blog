---
title: "Markdown is for the Web, AsciiDoc is for the Docs (and CVs)"
date: 2026-04-27
draft: false
tags: ["asciidoc", "markdown", "pdf", "tooling"]
categories: ["coding"]
description: "Why I moved my CV from Markdown to AsciiDoc to get the perfect PDF output."
---

We all love Markdown. It's the lingua franca of the modern developer—the default choice for READMEs, GitHub comments, and even this blog. But recently, I hit a wall with it while trying to polish my CV.

Markdown is fantastic for **structure**, but it’s historically weak on **presentation**, especially when it comes to converting to PDF.

### The Problem with Markdown PDFs

When you want to convert a Markdown file to a professional-looking PDF, you usually have two choices:
1. Use an IDE plugin that "prints" the preview (hard to automate, limited styling).
2. Use a tool like Pandoc (powerful, but often requires a heavy LaTeX dependency).

I wanted something that felt like code—version-controllable, themeable, and easy to run from a shell script—but with the precision of a real document processor.

### Enter AsciiDoc

While Markdown was designed to simplify HTML, **AsciiDoc** was designed from the ground up as a replacement for DocBook. It's a "semantic" markup language. It understands that a document has a header, a footer, and metadata.

Here’s why I made the switch for my CV:

#### 1. Native PDF Support (Asciidoctor PDF)
The Ruby-based `asciidoctor-pdf` tool is a game changer. It doesn't just "print a webpage"; it builds a PDF from a YAML-based theme file.

To get started on macOS, you can install the base processor via Homebrew and the PDF extension via RubyGems:

```bash
# Install the base processor
brew install asciidoctor

# Install the PDF extension
gem install asciidoctor-pdf
```

#### 2. Theming without the Pain
Instead of fighting with CSS or LaTeX templates, AsciiDoc uses a straightforward YAML theme. I wanted my name centered and large, my section headings underlined, and a nice "Page X of Y" footer.

In `cv-theme.yml`, it looks like this:

```yaml
heading:
  h1:
    font_size: 28
    align: center
  h2:
    font_size: 16
    border_bottom_width: 0.5
    border_bottom_color: DDDDDD
footer:
  recto:
    right:
      content: '{page-number} of {page-count}'
```

#### 3. Control over Layout
In Markdown, forcing a page break is a hack. In AsciiDoc, it's a first-class citizen:
`<<<` 
That’s it. Page broken exactly where you want it.

### Fonts

I wanted a more control over fonts and needed it to be print friendly - so I moved the fonts into a clean structure, mapped Roboto (my favorite clean sans-serif) into the AsciiDoc catalog, and iterated on the theme until the PDF looked exactly right. 

The directory containing the page looks like this:
```text
.
├── Paul-Rule-CV.pdf          <!-- the output PDF
├── cv-theme.yml              <!-- configuration for the theme
├── fonts
│  └── Roboto
│      ├── OFL.txt
│      ├── README.txt
│      ├── Roboto-Italic-VariableFont_wdth,wght.ttf
│      ├── Roboto-VariableFont_wdth,wght.ttf
│      └── static
│          ├── Roboto-Bold.ttf
│          ├── Roboto-BoldItalic.ttf
│          ├── Roboto-Italic.ttf
│          └── Roboto-Regular.ttf
├── generate-pdf.sh           <!-- script I can run to generate the PDF
└── index.adoc                <!-- page content as AsciiDoc
```

I've put the command to generate the PDF in a shell script `generate-pdf.sh`:

```bash
asciidoctor-pdf \
  -a pdf-theme=cv-theme.yml \
  -a pdf-fontsdir="fonts/Roboto/static;GEM_FONTS_DIR" \
  -o Paul-Rule-CV.pdf \
  index.adoc
```

### The Verdict

I’m keeping my blog in Markdown—Hugo is built for it, and it works perfectly for web content. But for anything that needs to leave the browser and live as a professional document? **AsciiDoc is the winner, hands down.**

Luckily Hugo supports both Markdown and AsciiDoc, so I can use the AsciiDoc version for both web and PDF - no need for different versions based on presentation mode.

If you're still fighting with Word or basic Markdown-to-PDF converters, give Asciidoctor a look. 

Here is the result: [PDF](/pauls-blog/page/cv/Paul-Rule-CV.pdf).
