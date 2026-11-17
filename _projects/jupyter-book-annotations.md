---
layout: page
title: Jupyter Book Annotations
description: A private, local Hypothesis-style plugin for Jupyter Book 2 and MyST that lets readers highlight, annotate and copy text, with everything stored in their own browser.
img: assets/img/jupyter-book-annotations.png
importance: 2
category: work
---

[View the source on GitHub →](https://github.com/gradill22/myst-annotate) &emsp; [Install from npm →](https://www.npmjs.com/package/myst-annotate)

## Abstract

Reading a textbook online should feel like reading one on paper: you should be able to grab a highlighter, scribble in the margins, and come back to your notes later. Hypothesis[^1] does this for the web, but it comes with accounts, servers, and social features that a reader studying on their own doesn't need.

`myst-annotate` is a plugin for Jupyter Book 2[^2] and any other MyST Markdown[^3] site. Readers can highlight text in five colours, write notes in Markdown with LaTeX math, and copy text, all without an account. There are no servers. Everything is stored in the reader's own browser.

The plugin is a single file that an author adds to their book's `myst.yml`. It works offline once the book is built, and it never modifies the page's HTML.

## Tools

* Node.js (22 and 24)
* Jupyter Book 2 with the MyST Markdown engine
* esbuild - Bundler
* Playwright - Browser testing
* GitHub Actions - CI and releases

### npm Packages

* `markdown-it` - Markdown rendering for notes
* `katex` - LaTeX math rendering for notes
* `esbuild` - Bundles the plugin and widget into single files
* `playwright` - Drives a real browser in the tests
* `jupyter-book` - Builds real books in the tests

### Browser Support

The plugin runs in any browser with the CSS Custom Highlight API[^4]:

* Chrome/Edge: 105+
* Safari: 17.2+
* Firefox: 140+

In older browsers, the plugin does nothing and logs a console warning.

## Setup

**Overview:** This plugin has two halves. The first is a MyST transform plugin that runs when the book is built: it adds a small widget to the end of every page. The second is the widget itself, which runs in the reader's browser. It draws highlights, shows the toolbars and the notes sidebar, and saves everything to the browser's `localStorage`. The widget bundles everything it needs, so a built book never has to reach the internet.

### MyST Plugins

MyST lets authors extend their books with plugins: JavaScript modules that add directives, roles, or transforms. Our plugin is a transform. At build time, it walks each page's document tree and appends an `anywidget` node pointing at our widget. MyST copies the widget file into the built site, and the book theme loads it in the browser.

Authors can also skip the plugin and embed the widget on chosen pages only with MyST's `anywidget` directive. This is handy for books that only want annotation on a few pages.

### Highlights

When a reader selects text, a toolbar pops up with **Annotate**, five colour swatches (yellow, magenta, cyan, green, and red), a clear swatch, a copy button, and a ✕ to close it.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/jupyter-book-annotations-toolbar.png" title="The selection toolbar" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The selection toolbar: Annotate, five colours, the clear swatch, Copy, and close.
</div>

Highlights can overlap. A new highlight is drawn on top of older ones, and removing it reveals the one underneath. Clicking a highlight brings the toolbar back to recolour it, remove it, copy it, or annotate it.

### Annotations

Choosing **Annotate** opens the sidebar with a new note attached to the selected text. Notes support Markdown and LaTeX math (`$…$` and `$$…$$`), with a Write/Preview toggle so readers can check their math before saving. Annotated text gets a dotted amber underline. Unsaved drafts are kept, so a half-written note comes back the next time the reader opens the page.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="lazy" path="assets/img/jupyter-book-annotations-note-write.png" title="Writing a note" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="lazy" path="assets/img/jupyter-book-annotations-note-preview.png" title="Previewing a note" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The same note in Write mode (left) and Preview mode (right).
</div>

Clicking annotated text opens a popover with the notes under that point. From there, readers can copy the quoted text, highlight it, edit the note, or delete it.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="lazy" path="assets/img/jupyter-book-annotations-popover.png" title="The annotation popover" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The popover for clicked annotated text, with colours for the highlight underneath.
</div>

### The Sidebar

The sidebar opens from a tab on the right edge of the page or with **Alt/⌥+Shift+A**. It has four tabs:

* **Annotations:** every note on the page.
* **Highlights:** every highlight, each with buttons to annotate, copy, or delete it.
* **Page notes:** notes about the whole page rather than a passage.
* **Orphans:** notes whose text is no longer on the page.

A **This page / Whole book** switch widens the view to every page of the book, grouped by chapter. There's also a search box and an eye button that hides or shows every mark at once.

<div class="row justify-content-sm-center">
    <div class="col-sm-6 mt-3 mt-md-0">
        {% include figure.liquid loading="lazy" path="assets/img/jupyter-book-annotations-whole-book.png" title="The Whole book view" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The Whole book view lists highlights from every chapter.
</div>

### Local Storage

All highlights and notes live in the browser's `localStorage`, with one entry per page. There are no accounts and nothing is ever sent anywhere. The trade-off is that a reader's notes stay on one browser on one device. For a private study tool, that's a feature, not a bug.

## Challenges

There were several challenges along the way when developing this plugin.

### Marking a React Page

The traditional way to highlight text on a web page is to wrap it in `<mark>` elements. The Jupyter Book theme is a React application, and React owns the page's HTML. If we start inserting elements into the page, React may throw them away the next time it renders, or worse, break when it finds HTML it didn't put there. The theme also renders widgets inside a shadow DOM, so our widget's styles can't reach the page's text.

### Anchoring Marks

A highlight needs to stay attached to the same words every time the reader comes back. But books change: an author fixes a typo, adds a paragraph, or rewrites a section, and the page's text shifts underneath the saved highlights. We need a way to find the right text again, and an honest answer when it's gone.

### One Self-Contained File

The very first version of this plugin was two files: the plugin and a separate script it loaded. When we listed the plugin in a book's `myst.yml` by URL, the build failed with `SyntaxError: Unexpected token '<'`. MyST downloads a plugin listed as a URL, but that URL served a web page (HTML) instead of JavaScript, and the built plugin wasn't published anywhere that served the file itself. And even with the right URL, MyST only downloads that one file, so the script it depended on would never arrive. On top of that, the notes need markdown-it, KaTeX, and KaTeX's fonts, and readers shouldn't need internet access to see their own math.

## Solutions

Given the challenges above, here are the solutions that got us past those barriers.

### The CSS Custom Highlight API

Instead of changing the page's HTML, the plugin uses the CSS Custom Highlight API[^4]. We describe each highlight as a `Range` of text, register the ranges with the browser, and style them with the `::highlight()` pseudo-element. The browser paints the colours on top of the text without adding a single element to the page, so React never notices. There is one registered highlight per colour. To layer overlapping highlights, the plugin works out which parts of each one are still visible under the newer ones and paints only those parts, so the newest highlight always sits on top.

The toolbars, popover, and sidebar live in their own shadow root, so the theme's styles don't leak into them. The few styles that must apply to the page itself, such as the highlight colours and KaTeX's fonts, are added to the document's head.

### Offsets with a Text Fallback

Each mark is saved with its character offsets within the page's `<article>` and a copy of the text it covers. When the page loads, the plugin first tries the saved offsets. If the text found there doesn't match the saved quote, it searches the page for the quote and moves the mark to wherever it's found. If the quote is gone entirely, the note becomes an orphan: it stays in the sidebar's **Orphans** tab instead of vanishing or attaching itself to the wrong text.

### Bundling Everything

We now use esbuild to bundle the widget, markdown-it, KaTeX, and KaTeX's fonts (embedded as data URLs) into one browser file. That file is then embedded inside the plugin itself. When MyST loads the plugin, it writes the widget next to itself under a content-hashed name, and MyST copies it into the built site. The result is a single `.mjs` file that works whether it's installed from npm, built from GitHub, or downloaded from a URL. The widget is about 750 KB, or about 400 KB gzipped, and the browser caches it after the first page.

Every GitHub release also attaches the built plugin, so authors have a URL that serves real JavaScript.

## Final Product

The plugin is published on npm as `myst-annotate`, and every GitHub release attaches the built plugin and widget. Releases are published to npm automatically from GitHub through trusted publishing, after the full test suite passes. That suite has 90 tests, including real Jupyter Book builds and Playwright browser tests, and runs on Linux and Windows with Node 22 and 24 for every pull request.

Adding it to a book takes one line in `myst.yml`:

```yaml
project:
  plugins:
    - https://github.com/gradill22/myst-annotate/releases/latest/download/myst-annotate.mjs
```

The plugin has also been submitted to the official Jupyter Book MyST plugins catalogue[^5].

If you want to try it on your own book, you can find installation instructions on [GitHub](https://github.com/gradill22/myst-annotate) and [npm](https://www.npmjs.com/package/myst-annotate).

<sub>Screenshots show the plugin on [Learn Probability](https://snowch.github.io/learn_probability/) by Chris Snow, licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The highlights and notes were added for these screenshots.</sub>

---

[^1]: https://web.hypothes.is/
[^2]: https://jupyterbook.org
[^3]: https://mystmd.org
[^4]: https://developer.mozilla.org/en-US/docs/Web/API/CSS_Custom_Highlight_API
[^5]: https://github.com/jupyter-book/myst-plugins/pull/52
