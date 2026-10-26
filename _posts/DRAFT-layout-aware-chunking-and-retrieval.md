---
layout: post
title: "Layout-aware chunking and retrieval"
excerpt: "The chunk that best matches a question is often not the one that answers it. The heading that says what it applies to, or the exception in the next paragraph, sits in a chunk that scored low. Layout-aware chunking rebuilds the document's outline from how it looks, and returns the whole section a match belongs to."
tweet:
published: false
---

Most retrieval pipelines cut a document into chunks, score each chunk against the question and hand the best ones to the model. That works until the answer depends on text the best chunk doesn't contain.

The short version: the chunk that best matches a question is often not the one that answers it. The heading that says what it applies to, or the exception in the next paragraph, sits in a chunk that scored low. Layout-aware chunking rebuilds the document's outline from how it looks, then returns the whole section a match belongs to, with every heading above it.

## The chunk that matches isn't the chunk that answers

Each chunk is scored on its own text. Whether it answers the question often depends on text somewhere else. That text scores low and is left out. The model gets a fragment that looks complete and answers confidently.

Here is a small refund policy, cut into fixed-size chunks.

```
chunk 1   Refund Policy
          Monthly plans
          Monthly plans can be cancelled at any time.
chunk 2   The current month is not refunded.
          Annual plans
          Within 30 days of purchase, annual plans are refunded in full.
chunk 3   After 30 days, refunds are prorated for the unused months.
chunk 4   Exception: plans bought at a promotional price are final sale.
          How to request
          Email billing@acme.example with your order number.
```

Ask it "Can I get a refund after 30 days?" and chunk 3 is the best match by a distance. The model answers "Yes, refunds after 30 days are prorated." That is wrong twice. The rule only applies to annual plans, and the heading that says so is in chunk 2. Plans bought at a promotional price can't be refunded at all, and that line is in chunk 4.

Raising the number of chunks you return doesn't fix this reliably. The next best chunk is chunk 2. It brings the "Annual plans" heading, but also the monthly rule "The current month is not refunded". Now the model sees text from two sections with no boundary between them. And the exception in chunk 4 still doesn't make the cut.

There are three ways a chunk loses what it needs.

1. **Lost parent.** The heading that scopes the passage is in another chunk. It might be a plan, a fiscal year or a version.
2. **Lost sibling.** The exception, the next row or the "however" paragraph is in the next chunk.
3. **Lost boundary.** A chunk spans two records and facts cross over from one to the other.

## Why splitting on headings isn't enough

The obvious fix is to split on headings instead of on size. Many libraries do this. It helps, but only if the headings can be found.

In HTML, a lot of headings are not heading tags. They are paragraphs made bigger and bolder with styling. When I checked a sample of a few hundred web pages, more than half of the headings were like that. PDFs are worse. They have no heading markup at all, only text drawn in different fonts at different sizes.

So a splitter that looks for heading markup sees one long section where a reader sees ten.

## Rebuilding the outline from the layout

A reader finds the headings by looking at the page. Short text that is bigger or bolder than the text around it is a heading. Bigger means higher up. That is all layout-aware chunking does.

1. **Recover the outline.** Render the HTML page in a browser, or read the fonts out of the PDF. Treat short text that is larger or bolder than the body text as a heading. Rank the headings by size, and hang every block of text under the nearest heading above it.
2. **Find the match.** Search the nodes of that tree for the question, by keywords or by embeddings.
3. **Expand along the tree.** Return the whole section the match belongs to, with its ancestor headings as a breadcrumb, such as `Refund Policy › Annual plans`.
4. **Deduplicate by node.** Keep track of which sections were already sent, so a follow-up question returns only new text.

The first step is the only unusual part, and it is shorter than it sounds.

```python
def build_tree(blocks, body_size):
    root = Section(level=0, title="")
    open_sections = [root]
    for block in blocks:                      # in reading order
        if is_heading(block, body_size):      # short, and bigger or bolder than body text
            level = heading_level(block.size) # bigger font, higher level
            while open_sections[-1].level >= level:
                open_sections.pop()
            section = Section(level, block.text)
            open_sections[-1].children.append(section)
            open_sections.append(section)
        else:
            open_sections[-1].children.append(Text(block.text))
    return root
```

For the refund question, the match is still the line about prorated refunds. What the model gets is different.

```
SECTION: Refund Policy › Annual plans
Within 30 days of purchase, annual plans are refunded in full.
After 30 days, refunds are prorated for the unused months.
Exception: plans bought at a promotional price are final sale.
```

The model now answers "On an annual plan, yes, prorated for the unused months, unless it was bought at a promotional price." That is the right answer.

The size of what comes back is no longer fixed in advance. It comes from the document and the question together. It might be one line, a section or a whole page. The parent heading always comes along, and a section never runs into the next one.

## When a section is too big

Returning the whole section works well when sections are small. Some aren't. A long list of products or a schedule with fifty rows can be one section. A match on one line shouldn't drag the whole thing in. We built two ways to handle that.

1. **A sibling window.** Return the matched line with the line before and the line after it under the same heading. It works like the sliding windows many pipelines already use, but it stops at the edge of the section instead of running into the next one. It is cheap, and it keeps what you send small. It suits long lists.
2. **Climb, then search inside.** After the first match, climb up the tree to the largest section around it that fits a size budget. Then search again, inside that section only. If no section fits the budget, fall back to the sibling window.

The second one fixes something the first can't. In the refund example, the line about promotional prices scores too low against the whole document to be returned. Inside "Annual plans", it only competes with two other lines, so it makes the cut. Text that looks unimportant against the whole document can be the second most relevant thing in its own section.

## Compared with other approaches

None of this is the only way to give a chunk more context. Here is how it compares with the approaches I see most often.

| Approach | Where chunks are cut | What comes back |
|---|---|---|
| Fixed-size windows | Every so many tokens | The matching chunks |
| Splitting on headings | Heading markup, when there is any | The matching chunks |
| Contextual retrieval | Fixed size, each with a model-written summary of where it sits | The matching chunks |
| Parent-document retrieval | Small chunks inside bigger fixed-size ones | The bigger chunk around the match |
| Layout-aware chunking | The document's outline, rebuilt from its layout | The matched section, with its headings |

Parent-document retrieval shares the idea of matching small and returning bigger. Its parent is a fixed-size block, though, so it can cut a section in half or join two sections together. Contextual retrieval puts the parent heading back, but it costs a model call for every chunk, and siblings still end up in other chunks. Layout-aware chunking returns the unit the author wrote.

## Where it came from

This came out of a real failure, in a system much like the contract extraction I described in the [first post](/when-one-agent-has-to-do-six-jobs/).

Picture a framework agreement with several supplier schedules back to back. Each schedule has its own payment terms, under a heading that is just bold text. We asked an agent for one supplier's payment terms. The chunk it got started halfway through the previous schedule, so it reported the wrong supplier's terms as the right one's. That is a lost boundary, and nothing in the chunk said so.

Rebuilding the headings from the layout fixed most of it. In a test on PDFs, the share of reads that mixed up two records went from most of them to fewer than one in ten.

It also helps the agents I wrote about earlier in this series. In the first post, a discovery agent finds the parts of a contract that matter, and I noted that those parts rarely make sense on their own. In the [second post](/agents-calling-agents-in-one-process/), a document worker reads whatever the user uploads. Both do better when the text they get comes with the headings that say what it is about.

## Limits

It is not free, and it doesn't fix everything.

1. **It needs a layout.** Plain text has no fonts or sizes to read. Scanned PDFs need the page images read first.
2. **Rendering costs time.** Rendering an HTML page in a browser is slower than reading its text.
3. **Some documents are badly laid out.** If the author made body text bold for emphasis, it can look like a heading. Short and bold is a good rule, not a perfect one.

If your answers are often right in what they say but wrong about what it applies to, look at what your chunks lose before you look at your prompts.
