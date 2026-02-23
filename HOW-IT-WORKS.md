# How the Three-Column Layout Works

This document explains the techniques used to turn a single-column Markdown page into a three-column layout inside a Material for MkDocs site. The goal is to give you enough understanding of each tool that you could build something similar from scratch—or adapt this pattern to other pages.

---

## The problem to solve

The page had a repeating structure: eight major sections, each containing several subsections of prose, followed by a block of learning resources. On a normal single-column page, a reader who cares about a section's resources has to scroll far past the prose to find them, or keep the page open in two browser tabs. The ideal layout presents the prose and its resources side-by-side, at the same scroll position.

To do that, you need three things:

1. **A way to define a multi-column layout in CSS** — CSS Grid
2. **A way to keep side panels fixed in view while the main content scrolls** — `position: sticky`
3. **A way to put HTML layout containers inside a Markdown file** without losing Markdown formatting inside them — the `md_in_html` extension

All three are explained below, then we'll walk through how they're connected.

---

## Tool 1: Material for MkDocs customization hooks

Material for MkDocs has several ways to inject your own code without modifying the theme files:

### `extra_css` in `mkdocs.yml`

```yaml
extra_css:
  - stylesheets/three-column.css
```

This tells MkDocs to load `docs/stylesheets/three-column.css` on every page, after Material's own stylesheets. Because it loads after, your rules take precedence over the theme's defaults when the specificity is equal. The path is relative to the `docs/` directory.

### `markdown_extensions` in `mkdocs.yml`

```yaml
markdown_extensions:
  - md_in_html
  - attr_list
```

These are Python-Markdown extensions that MkDocs passes to its Markdown parser. They add capabilities to the Markdown language itself. More on these below.

### Front matter: `hide` keys

At the top of the Markdown file, a YAML block between `---` lines controls per-page Material settings:

```yaml
---
hide:
  - toc
  - navigation
---
```

`hide: [toc]` suppresses Material's automatically-generated right-sidebar table of contents for this page. Normally Material scans your headings and renders a floating ToC on the right. Since we're building our own ToC column on the left, the auto-generated one would be redundant and would conflict with the layout. `hide: [navigation]` removes the left site-wide navigation sidebar, freeing up the full page width for our three-column layout.

These `hide` keys are a Material-specific feature—they're not standard MkDocs.

---

## Tool 2: The `md_in_html` extension

By default, if you write HTML tags in a Markdown file, Python-Markdown treats everything inside those tags as raw text—it won't parse Markdown syntax inside HTML blocks. That means this would render the headings literally, not as `<h3>` elements:

```html
<div class="section-main">

### Description

Some text here.

</div>
```

The `md_in_html` extension changes this behavior. When you add the `markdown` attribute to an HTML element, the extension processes the contents as Markdown:

```html
<div class="section-main" markdown>

### Description

Some text here.

</div>
```

**The blank lines are critical.** There must be a blank line after the opening tag and before the closing tag. Without them, the extension's parser doesn't recognize the content as a Markdown block and leaves it as-is. This is the single most common mistake when using this extension.

The `attr_list` extension (the other one added to `mkdocs.yml`) allows adding CSS classes and IDs to Markdown elements using `{: }` syntax, e.g., `## Heading {: #my-id .my-class }`. It's used less directly in this layout, but it's a companion extension that `md_in_html` depends on internally.

---

## Tool 3: CSS Grid

CSS Grid is a two-dimensional layout system. Before it existed, multi-column layouts required floats or flexbox tricks that were fragile and hard to reason about. Grid makes column-and-row layouts explicit and predictable.

### The basics

```css
.container {
    display: grid;
    grid-template-columns: 240px 1fr;
    gap: 2rem;
}
```

- `display: grid` — turns the element into a grid container. Its direct children become grid items, and each one goes into a cell.
- `grid-template-columns: 240px 1fr` — defines two columns. The first is exactly 240px wide. The second is `1fr`, which means "one fraction of the remaining space." After the 240px column and the gap are accounted for, `1fr` takes everything left.
- `gap: 2rem` — the space between columns (and rows, if there are multiple rows).

You can have as many columns as you want:

```css
grid-template-columns: 240px 1fr 300px; /* three columns */
```

### Why we used two nested grids instead of one three-column grid

The obvious approach would be a single three-column grid: `[ToC] [Content] [Resources]`. But there's a constraint: each section's resources panel should only be sticky *within that section's height*. If all resources were siblings in a single flat grid, sticky positioning would keep them all visible at once, which makes no sense.

Instead, the layout uses two nested grids:

**Outer grid** — divides the page into a left ToC column and a "rest of the page" column:
```
[.toc-left (240px)] [.sections-wrapper (1fr)]
```

**Inner grid** — inside `.sections-wrapper`, each section is its own grid dividing into content and resources:
```
[.section-main (1fr)] [.section-resources (300px)]
```

This nesting means each `.section-resources` panel is contained within its own `.section-grid`, so sticky positioning works correctly per-section. When the section scrolls out of view, that section's resources panel goes with it.

### Spanning across columns

The `<h2>` heading for each section needs to appear above both the content and resources columns—it should span the full width of its section. CSS Grid has a property for this:

```css
.section-heading {
    grid-column: 1 / -1;
}
```

`grid-column: 1 / -1` means "start at column line 1, end at the last column line." The `-1` always refers to the last line regardless of how many columns there are. This is why the `<h2>` is written as raw HTML rather than Markdown syntax:

```html
<h2 id="security-governance-principles" class="section-heading">
    Security Governance Principles
</h2>
```

A Markdown `## Heading` would end up inside `.section-main` (because of how the divs are nested), not as a direct child of `.section-grid`. Only direct children of a grid container become grid items—you can't apply `grid-column` to a grandchild. Writing the `<h2>` as raw HTML at the right level in the div structure lets us give it the class and have it sit directly inside `.section-grid`.

---

## Tool 4: `position: sticky`

`position: sticky` is a hybrid positioning mode. An element with `position: sticky` behaves like a normal in-flow element—it takes up space in the document and scrolls with the page—until it would scroll past a threshold you set with `top`, `bottom`, `left`, or `right`. At that point it "sticks" and stays visible, as if it were `position: fixed`, until its parent container scrolls out of view.

```css
.section-resources {
    position: sticky;
    top: var(--md-header-height, 5rem);
    max-height: calc(100vh - var(--md-header-height, 5rem) - 2rem);
    overflow-y: auto;
}
```

Breaking this down:

- `top: var(--md-header-height, 5rem)` — the panel sticks when its top edge reaches this distance from the viewport top. The `var(...)` syntax reads a CSS custom property (more on those below). The `, 5rem` is a fallback value if the property isn't defined.
- `max-height: calc(...)` — limits how tall the panel can be. Without this, a very long resources list would overflow the viewport. `calc()` lets you do arithmetic with mixed units: `100vh` is the full viewport height, minus the header height, minus a small margin.
- `overflow-y: auto` — when the panel's content is taller than `max-height`, show a scrollbar inside the panel rather than overflowing into the page.

### The containment rule: why this works per-section

Here's the key behavior of `position: sticky`: **a sticky element can only move within its containing block.** Once its parent container scrolls out of view, the sticky element goes with it. It will never escape its parent.

This is exactly what we want. Each `.section-resources` is inside its own `.section-grid` container. The resources panel sticks while you scroll through that section. Once the `.section-grid` container scrolls out of view, the resources go with it, and the next section's resources panel takes over. No JavaScript needed.

### The left ToC column works the same way

```css
.toc-left {
    position: sticky;
    top: var(--md-header-height, 5rem);
    max-height: calc(100vh - var(--md-header-height, 5rem) - 2rem);
    overflow-y: auto;
}
```

The ToC is a direct child of `.page-three-col`, which is inside the page's full content area. Since the page content area is effectively the full document height, the ToC sticks for the entire scroll length of the page.

---

## CSS custom properties from the Material theme

Rather than hard-coding colors or pixel values that might not match the theme, the CSS reads values that Material defines:

| Property | What it is |
|---|---|
| `--md-header-height` | The height of the top navigation bar |
| `--md-code-bg-color` | The background color used for code blocks |
| `--md-default-fg-color--light` | A lighter variant of the main text color |
| `--md-accent-fg-color` | The accent/highlight color from the theme palette |
| `--md-default-fg-color--lightest` | A very faint variant used for borders and dividers |

These are CSS custom properties (also called CSS variables), defined by Material using `--` prefixed names. When Material switches between light and dark mode, it redefines these variables, and your styles automatically follow. You don't need separate `@media (prefers-color-scheme: dark)` rules.

---

## The `:has()` selector for scoped max-width

Material's content area has a default max-width (~61rem) that centers text on wide screens—good for readability in single-column prose, but constraining for a multi-column layout. We need to remove it, but only on pages that use our layout, not globally.

```css
.md-content__inner:has(.page-three-col) {
    max-width: none;
}
```

`:has()` is a CSS selector that matches an element *if it contains a descendant matching the inner selector*. So this rule reads: "Apply `max-width: none` to `.md-content__inner` only when it contains a `.page-three-col` descendant." Pages without the layout wrapper are unaffected. This is a relatively new CSS feature (Safari 15.4+, Chrome 105+, Firefox 121+), which covers all modern browsers.

---

## Responsive design: what happens at smaller screen widths

Two `@media` rules handle narrower viewports:

### Tablet (~769px to 1219px)

```css
@media (max-width: 1219px) {
    .page-three-col {
        grid-template-columns: 1fr; /* ToC and sections stack */
    }
    .toc-left {
        position: static; /* no longer sticky */
        border-bottom: 1px solid var(--md-default-fg-color--lightest);
    }
    .section-grid {
        grid-template-columns: 1fr 280px; /* sections keep two columns */
    }
}
```

The outer grid collapses to one column (ToC above sections), but each section's content + resources side-by-side layout is preserved.

### Mobile (below 768px)

```css
@media (max-width: 767px) {
    .section-grid {
        grid-template-columns: 1fr; /* everything stacks */
    }
    .section-resources {
        position: static;
        max-height: none;
    }
}
```

Everything flows vertically. Sticky positioning is removed from resources since there's no adjacent column to stick alongside. On a narrow screen, reading prose then resources below is natural.

---

## How the pieces connect: the full HTML structure

Here's the skeleton of the Markdown file with the HTML wrappers, stripped of content:

```
---
hide:
  - toc
  - navigation
---

# Page Title

<div class="page-three-col" markdown>         ← outer grid (2 columns)

  <div class="toc-left" markdown>             ← left column: sticky ToC
    - [Link](#anchor)
    ...
  </div>

  <div class="sections-wrapper" markdown>     ← right column of outer grid

    <div class="section-grid" markdown>       ← inner grid (2 columns, per section)

      <h2 id="slug" class="section-heading">  ← spans both inner columns
        Section Name
      </h2>

      <div class="section-main" markdown>     ← inner left: prose
        ### Description
        ...
      </div>

      <div class="section-resources" markdown> ← inner right: sticky resources
        ### Learning Resources
        ...
      </div>

    </div>  ← end section-grid

    ...more section-grids...

  </div>  ← end sections-wrapper

</div>  ← end page-three-col
```

Each level of `<div markdown>` is processed by `md_in_html`, so all the `###` headings, `-` lists, and `[links](url)` inside them render normally. The CSS Grid declarations on `.page-three-col`, `.section-grid`, etc. then place each div into its corresponding column.

---

## Summary: the four techniques and why each is necessary

| Technique | Why it's needed |
|---|---|
| `md_in_html` extension | Allows Markdown syntax inside HTML `<div>` elements—without it, headings and links inside the layout divs would appear as literal text |
| `hide: [toc, navigation]` front matter | Removes Material's built-in sidebars on this page, freeing the full width and preventing conflict with the custom ToC column |
| CSS Grid | Defines explicit column widths and places elements into them—far cleaner than float or flexbox hacks for this kind of structural layout |
| `position: sticky` | Keeps side panels in view while adjacent content scrolls, without any JavaScript; the containment rule (sticky elements can't escape their parent) is what makes per-section resources work correctly |
