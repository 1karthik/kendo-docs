---
title: Drawing HTML (DOM)
page_title: Draw a piece of HTML available in the DOM
position: 51
---

# HTML-drawing API

Using the `drawing.drawDOM` function you can draw a DOM element into a [drawing.Group](/api/dataviz/drawing/group), which you can then render with one of the supported backends into SVG, PDF, HTML5 `<canvas>` or VML.

The DOM element must be appended to the document and must be visible (i.e. you cannot draw an element which has `display: none` or `visibility: hidden`, etc.).  For example if you have this HTML in the page:

    <div id="drawMe" class="...">
      ... more HTML code here...
    </div>

You can draw it from JavaScript with the following call:

    drawing.drawDOM("#drawMe").then(function(group){
        // here group is a drawing.Group object

        // you can now draw it to SVG for example:
        var svg = drawing.Surface.create($("#container"), { type: "svg" });
        svg.draw(group);

        // or you can save it as PDF.
        // optionally:
        group.options.set("pdf", {...pdf options...});
        drawing.pdf.saveAs(group, "filename.pdf", proxyUrl);
    });

`drawing.drawDOM` takes a jQuery selector or object, or a plain DOM node, and returns a promise which will deliver a `drawing.Group` object.


### Custom fonts and PDF

If you need PDF output, for optimal layout and Unicode support your document should declare the fonts that it uses using [CSS `font-face` declarations](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face).  Starting with version Q3-2014-SP1, the PDF generator in Kendo UI is able to dig such declarations directly from the CSS and there is no need for you to manually call [pdf.defineFont](/framework/drawing/pdf-output.html#using-custom-fonts).

Here is an example CSS declaration:

    @font-face {
      font-family: "DejaVu Sans";
      src: url("fonts/DejaVu/DejaVuSans.ttf") format("truetype");
    }

Next, to make sure the elements that you are trying to export are using this font, simply specify `font-family: "DejaVu Sans"` in their styles.  For instance, to make all Kendo widgets use this font:

    .k-widget {
      font-family: "DejaVu Sans";
    }

Notes:

- the PDF generator supports only TrueType fonts with Unicode mappings.

- in order for automatic font discovery to work your CSS must reside on the same domain as the web page.

- Kendo UI bundles the DejaVu font family and will fall back to it for a few names like “Times New Roman”, “Arial”, “Courier”, or generics like “serif”, “sans-serif”, “monospace”, if no alternate fonts are specified.  This is so that Unicode works by default.  Note, however, that the layout problem will remain — the PDF output will be slightly different from the browser unless the exact same fonts are used.


### Multi-page PDF output

`drawing.drawDOM` allows you to create a [multi-page PDF](/framework/drawing/pdf-output.html#multiple-pages-output) by specifying manual page breaks.  For this, pass a second options argument which contains `forcePageBreak` — a CSS selector.  Here is an example which draws a grid on a multi-page PDF:

    <div id="grid"></div>

    <script>
      var data = [];
      for (var i = 1; i < 50; ++i) {
        data.push({ title: "Item " + i, id: i });
      }

      $("#grid").kendoGrid({
        dataSource: data,
        rowTemplate: $("#rowTemplate").html()
      });

      kendo.drawing
        .drawDOM("#grid", { forcePageBreak: ".page-break" })
        .then(function(group){
          drawing.pdf.saveAs(group, "multipage.pdf")
        });
    </script>

    <script id="rowTemplate" type="x/kendo-template">
      <!-- to every tenth row we add the "page-break" class -->
      <tr data-uid="#= uid #" class="#= (id%10 == 0 ? 'page-break' : '') #">
        <td>#: title #</td>
        <td>#: id #</td>
      </tr>
    </script>


### Automatic page breaking (Q1 2015)

When your target is PDF output, you can solicit automatic page breaking (subject to a few limitations detailed below) by informing `drawDOM` what is the desired page size and margins.  The options are `paperSize` and `margin`, same as documented in [PDF options](/framework/drawing/pdf-output.html#pdf-options).  You can still use `forcePageBreak` in this case to manually specify break points.

Example:

    <div id="grid"></div>
    <script>
      var data = [];
      for (var i = 1; i < 200; ++i) {
        data.push({ title: "Item " + i, id: i });
      }

      $("#grid").kendoGrid({ dataSource: data });

      drawing.drawDOM("#grid", {
        paperSize: "A4",
        margin: "2cm"
      }).then(function(group){
        drawing.pdf.saveAs(group, "grid.pdf");
      });
    </script>

#### Page break limitations

Page breaking happens only inside text nodes.  Therefore, if an element has no text content, it cannot be split across pages.  For example an `<img>`, or a `<div>` which has perhaps a border or background image, but otherwise no content, will not be split.  If such elements fall on a page boundary, they will be moved to the next page (along with all the following DOM nodes), and if they don't fit on a single page they will be truncated.

Here is the current list of nodes that will not be split: `<img>`, `<tr>`, `<iframe>`, `<svg>`, `<object>`, `<canvas>`, `<input>`, `<textarea>`, `<select>`, `<video>`.

Automatic page breaking cannot occur inside a positioned element (`position: absolute`).  Moreover, elements with `position: fixed` are not supported at all (most likely they will not show up in the output).  Such elements will simply be skipped over by the algorithm. For example, for the following input:

    <p>Foo</p>
    <div style="position: absolute; top: 1000px">Bar</div>
    <p>Baz</p>

on an A4 page, the output will just display the Foo and Baz paragraphs.  The positioned `<div>` would appear on the first page at height 1000, but that's beyond the page boundary so it will be clipped.

If the algorithm decides to move a node to the next page, all the DOM nodes following it will be moved as well, even if there might potentially be room for some of them on the current page.  An example where this happens is with floating elements:

    <p>
      some text before
      <img style="float: left" ... />
      some text after
    </p>

It can happen that this element ends up in a position where all the text fits on current page, but the image is higher and would fall on the boundary.  In this case, the image and “some text after” will move to the next page.


### Page template (headers and footers)

When multi-page output is requested (via `forcePageBreak`/`paperSize`) you can additionally specify a page template. This template will be inserted into each page before producing the output.  Via CSS it can easily be positioned relatively to the page.  The template can be a function, or a Kendo template, and it receives the number of the current page and the total number of pages.  Example:

    <script type="x/kendo-template" id="page-template">
      <div class="page-template">
        <div class="header">
          <div style="float: right">Page #:pageNum# of #:totalPages#</div>
          This is a header.
        </div>
        <div class="footer">
          This is a footer.
          Page #:pageNum# of #:totalPages#
        </div>
      </div>
    </script>

    <div id="grid"></div>

    <style>
      /* make sure everything in the page template is absolutely positioned.
       * since the pages will be embedded in an element with position: relative,
       * all positions here are actually relative to the page element.
       */
      .page-template > * {
        position: absolute;
        left: 20px;
        right: 20px;
        font-size: 90%;
      }
      .page-template .header {
        top: 20px;
        border-bottom: 1px solid #000;
      }
      .page-template .footer {
        bottom: 20px;
        border-top: 1px solid #000;
      }
    </style>

    <script>
      $("#grid").kendoGrid(...);
      drawing.drawDOM("#grid", {
        paperSize: "A4",
        margin: "3cm",
        template: $("#page-template").html()
      }).then(function(group){
        drawing.pdf.saveAs(group, "filename.pdf");
      });
    </script>


### Known limitations

- no rendering of shadow DOM

- no CSS box-shadow, text-shadow, gradients

- only `solid` border-style

- the content of the following elements is not rendered: `<iframe>`, `<svg>`, `<input>`, `<textarea>`, `<select>`.  A `<canvas>` will be rendered as an image, but only if it's “non-tainted” (does not display images from another domain).

- images hosted on different domains might not be rendered, unless permissive [Cross-Origin HTTP headers](https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_enabled_image) are provided by the server.  Similarly, fonts might not be possible to load cross-domain.

- when the generated document is opened with Acrobat Reader and you try to use the `Save As` option from the file menu an error is thrown.
`"The document could not be saved. There was a problem reading(23)"`. The solution is to open Acrobat Reader options (Edit → Preferences) and in the "Documents" section uncheck “Save As optimizes for Fast Web View”, which is enabled by default. After this, Save As will work without errors.


## Supported browsers

The HTML renderer has been tested in recent versions of Chrome, Firefox, Safari, Blink-based Opera, Internet Explorer 9 or later.

Internet Explorer <= 8 is not supported.
