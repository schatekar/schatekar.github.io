---
layout: post
title: "Shrink the page, keep the tree"
excerpt: "A web page has to be cut down before a model reads it. The usual ways of doing that throw away the structure that tells the model which facts belong together. This is a way to shrink the page and keep the tree. The same idea works for the JSON an agent reads back from its tools."
tweet:
published: false
---

Many agent workflows start with a web page. You fetch it, hand it to a model and ask for the facts on it. Pages are big, and the model pays for every token of them.

The short version: cleaning a page for a model is not only about making it smaller. The structure that tells the model which facts belong together lives in the HTML tree, and the usual ways of shrinking a page throw it away. We keep the tree, remove everything that carries no meaning and mark where each record ends. The same idea works for the JSON an agent reads back from its tools.

## The setup

We built a pipeline that reads supplier websites and pulls out a row for each product: its name, its price band and a link to its page. Each page goes to the model in one call. We don't chunk it, because a catalogue split across chunks is a catalogue the model sees in pieces. I wrote about what to do when you do have to cut a document up in [layout-aware chunking and retrieval](/layout-aware-chunking-and-retrieval/). The example in this post is a close parallel to a production extraction system I built.

That leaves one decision that matters more than the prompt does: what form the page takes when it reaches the model. The two usual answers both cost us accuracy.

1. **Raw HTML** keeps everything, including scripts, styles, tracking attributes and a cookie banner. On some pages the consent banner alone was about three quarters of the visible text.
2. **Markdown or plain text** is small, but it throws away the one thing the model most needs on a catalogue page: which pieces of text belong to the same product.

This post describes the middle path we use instead. Keep the HTML element tree, strip away everything that doesn't carry meaning and mark where each record ends. Then apply the same idea to JSON.

## What flattening breaks

Catalogue pages usually show products as cards. One element wraps a name and a price band, and the styling puts them together on screen. Convert the page to markdown and the wrapper disappears. The name and the band become two separate headings, and the band now sits between two names.

![On the left, a page flattened to markdown. The Mid band sits between Trail Pack 30L and Summit Jacket, and the model reports Summit Jacket as Mid. On the right, compacted HTML where each card is wrapped in its own element, and the model reports Summit Jacket as Premium](/images/shrink-the-page/1-flatten-cards.png)

Rewording the prompt didn't fix it, and neither did a higher reasoning effort. Changing the input did.

Tables fail in a quieter way. A coverage table lists suppliers down the side and regions across the top, and an empty cell means "not covered". A cleaner that removes empty elements removes those cells too. Every cell after the gap shifts left, and a supplier that covers the third region is read as covering the first.

![A coverage table with empty cells removed, where Halden's Primary has shifted left under North, so the model reads Halden as primary for North. The same table with empty cells kept, where Primary stays under South](/images/shrink-the-page/2-empty-cells.png)

We had seen wrong-region answers like this. We reproduced them on a made-up table, and this was the cause.

In both cases the information is in the page, but it is carried by the structure rather than the text. A format that keeps only the text can't keep it.

## The method

The cleaner is one pass over a parsed document. We use BeautifulSoup. Each step removes weight or makes a boundary visible, and none of them changes how elements nest.

![Six steps: remove non-content, isolate the body carefully, remove consent banners, keep a few attributes, drop empties and wrappers, and minify and mark boundaries](/images/shrink-the-page/3-method.png)

### Banners go first, while their names are still there

Banner removal runs before attributes are removed, because a consent banner is easiest to identify by its own class and id. We check two ways, since neither catches every banner on its own.

1. **Known signatures.** If an element's id, class, role or aria-label contains a known consent vendor's name, we remove the element and everything inside it. Adding a new vendor means adding one string to a list.
2. **Phrase density.** Some banners carry no telling attributes. For those, we start at a consent phrase such as "we use cookies" and walk up through its ancestors. We keep the highest ancestor where consent phrases are still dense: at least two different phrases, making up about a tenth of its text. When an ancestor also holds real content, the density drops below the threshold, so the walk stops before it reaches the page body.

### Attributes: keep the few that are content

Most attributes exist for styling or tracking and cost tokens for nothing. We keep these:

1. `href` on links, which become product URLs.
2. `src`, `srcset`, lazy-load sources and `alt` on images, because a product photo URL is an output field.
3. `id` on structural elements, which often names a product or a section, such as `#p-7`.
4. `role`, `aria-controls` and `data-tab`, which say there is content behind a tab or a filter.

Links and image sources are made absolute first, so any URL the model copies out still resolves.

### Boundaries a chunker or a model can see

The last step adds a blank line after `</a>`, `</li>`, `</tr>`, `</p>`, `</h1>` to `</h3>`, `</section>` and `</article>`. We deliberately leave out `</h4>` to `</h6>` and `</div>`, because inside a card those usually separate a name from its own price band. The result reads as one card, one row or one section per paragraph. That helps the model, and it means any paragraph-based splitter downstream keeps whole records.

```python
# the boundary step, in full
BOUNDARY = re.compile(r"(</(?:a|li|tr|p|h1|h2|h3|section|article)>)", re.I)

def clean_for_reader(html, base_url=None):
    return BOUNDARY.sub(r"\1\n\n", clean_html(html, base_url=base_url))
```

### What to keep

A cleaner that removes everything empty or decorative goes too far. Four things need to stay.

1. **Empty table cells.** Position carries meaning in a matrix. Rows and tables that are empty all the way through still go.
2. **Empty panels with controls.** A blank tab panel that a click would fill means there is more to fetch. Keep it with its control.
3. **Content outside the body.** One page put a 120-product list before a stray `<body>` tag. Isolating the body cut the page from 700 KB to 6 KB and took the list with it. Now that step is skipped when the body holds under half the text.
4. **Ids on wrappers.** A wrapper is only unwrapped if it has no id and no interaction attributes. An id is often a card's only label.

The first and third were each added after a page lost products.

## When the catalogue isn't in the markup at all

Some sites send their products in the HTTP response but never as HTML. The list sits in a script as `var products = [...]`, or in a large `data-*` attribute that a front-end framework reads at runtime. The cleaner correctly discards both, and with them the only copy of the catalogue. The fix is to turn that JSON back into structured HTML and append it to the cleaned page.

1. **Find carriers.** `data-*` attributes and scripts of a few thousand characters that parse as JSON.
2. **Walk the tree for products.** Look for arrays of several objects that have a name plus a price, a link or a SKU. The second field keeps lists of tags from qualifying.
3. **Emit only identity fields.** One table row per product. Embedded descriptions are left out, and the product link is followed later.
4. **Fall back to embedded markup.** With no clean records, append long HTML strings from the payload that name product fields, up to a size cap.

![A product list of 180 objects inside a script, 810 KB. Name, SKU and URL are kept, and description and images are dropped. It is appended to the page as a table of 30 KB with one row per product](/images/shrink-the-page/4-json-in-script.png)

Two guards keep this away from pages that already work. It only runs when the cleaned page has just a handful of product words such as price, SKU or in stock, and its output is kept only if it names more of them than the page already does.

## The same idea for the JSON the model reads back

Agents that run code read the result back as JSON, and a tool result stays in the conversation, so its size is paid again on every later call. We compared three ways of serialising the same results.

1. **Indented JSON** was about 80% larger than the alternatives.
2. **Fully compact JSON** was the smallest, but it puts a long array on one line. Small models then misread where one record ends and the next begins.
3. **One record per line** uses compact separators, with each record on its own line. It was about 45% smaller than indented JSON, under 1% larger than fully compact, and read most accurately on large tables.

![Three bars for the same tool result. Indented JSON is the longest. Fully compact is about half the size, but small models lose where a record ends. One record per line is about the same size as compact and reads most accurately](/images/shrink-the-page/5-serialising.png)

```python
def one_per_line(obj):
    # a list of records: one compact record per line
    if isinstance(obj, list) and obj and all(isinstance(x, dict) for x in obj):
        return "[\n" + ",\n".join(compact(x) for x in obj) + "\n]"
    # a dict: recurse, so nested record lists get the same treatment
    if isinstance(obj, dict):
        return "{" + ",".join(json.dumps(str(k)) + ":" + one_per_line(v)
                              for k, v in obj.items()) + "}"
    return compact(obj)  # json.dumps(obj, separators=(",", ":"))
```

When a result is still too large, a cut by character count leaves half a record. We do three things instead.

1. **Keep whole items and say how many.** "Showing 120 of 180 items."
2. **Show part of the first item that didn't fit.** A large nested field can't vanish entirely that way.
3. **Describe what was dropped by its shape alone.** The model can then ask for exactly the fields it needs, without first spending a call to explore.

```
{"products":[
{"name":"Trail Pack 30L","band":"Mid"},
{"name":"Summit Jacket","band":"Premium"}
],"total":180}

showing 120 of 180 items; dropped 60 items
shaped like {name: str, band: str}
```

## Results

On ordinary pages, token cost came out about the same as markdown. The HTML tags we keep cost roughly what the removed boilerplate saved. The gain is in accuracy, not price.

1. The price band mix-up was fixed on the page where we first saw it.
2. The banner filter wrongly stripped nothing from any of the 140 pages we checked.
3. The largest reduction in text was about three quarters, on a page dominated by a consent banner.
4. Recovery from embedded data found 180 of 180 products on the test page, matching a hand-checked count. On three pages that already rendered their catalogue correctly, it left them untouched.
5. Tool results came out about 45% smaller with one record per line than with indented JSON.

## Where it falls short

1. Div-only cards with no link, list item or row around them get no boundary break. The model can still follow the nesting. A paragraph splitter can't.
2. New banner vendors and unusual consent wording get through until they are added.
3. Data loaded after the page isn't in the page at all. That needs a render wait or a simulated click.
4. Unusual JSON keys are missed until a synonym for name, price or link is added.

Decide what the model needs to know about how facts relate, and make sure your input format still carries it.

On catalogue pages, that relationship lives in the element tree and the positions of table cells, and sometimes in JSON the page never displays. Compaction removes everything around that structure and leaves the structure itself alone.
