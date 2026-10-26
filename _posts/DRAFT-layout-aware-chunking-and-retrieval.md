---
layout: post
title: "Layout-aware chunking and retrieval"
excerpt: "The chunk that best matches a question is usually relevant, but it may not be enough to answer it fully. The heading that says what it applies to, or an exception in the next paragraph, can sit in a chunk that scored low. That text can complete the answer or change it entirely. Layout-aware chunking rebuilds the document's outline from how it looks, and returns the whole section a match belongs to."
tweet:
published: false
---

Most retrieval pipelines cut a document into chunks, score each chunk against the question and hand the best ones to the model. That works until the answer depends on text the best chunk doesn't contain.

The short version: the chunk that best matches a question is usually relevant, but it may not be enough to answer it fully. The heading that says what it applies to, or an exception in the next paragraph, can sit in a chunk that scored low. That text can complete the answer or change it entirely. Layout-aware chunking rebuilds the document's outline from how it looks, then returns the whole section a match belongs to, with every heading above it.

## The chunk that matches isn't always the whole answer

Each chunk is scored on its own text. The full answer often depends on text somewhere else, and that text can score low and be left out. The model gets a fragment that looks complete and answers confidently.

Here is a small refund policy, cut into fixed-size chunks. On the right is what this post ends up returning instead.

![On the left, a refund policy cut into four fixed chunks. The best match is chunk 3, while the "Annual plans" heading sits in chunk 2 and the promotional exception in chunk 4. On the right, the same policy as a tree, where the whole "Annual plans" section is returned around the match](/images/layout-aware-chunking/1-chunks-vs-tree.png)

Ask it "Can I get a refund after 30 days?" and chunk 3 is the best match by a distance. The model answers "Yes, refunds after 30 days are prorated." That is wrong twice. The rule only applies to annual plans, and the heading that says so is in chunk 2. Plans bought at a promotional price can't be refunded at all, and that line is in chunk 4.

Raising the number of chunks you return doesn't fix this reliably. The next best chunk is chunk 2. It brings the "Annual plans" heading, but also the monthly rule "The current month is not refunded". Now the model sees text from two sections with no boundary between them. And the exception in chunk 4 still doesn't make the cut.

There are three ways a chunk loses what it needs.

1. **Lost parent.** The heading that scopes the passage is in another chunk. It might be a plan, a fiscal year or a version.
2. **Lost sibling.** The exception, the next row or the "however" paragraph is in the next chunk.
3. **Lost boundary.** A chunk spans two records and facts cross over from one to the other.

## Why splitting on headings isn't enough

The obvious fix is to split on headings instead of on size. Many libraries do this. It helps, but only if the headings can be found.

In HTML, a lot of headings are not heading tags. They are paragraphs made bigger and bolder with styling. When I checked a sample of a few hundred web pages, more than half of the headings were like that. PDFs are worse. They have no heading markup at all, only text drawn in different fonts at different sizes.

So a splitter that looks for heading markup sees one long section where a person reading the page sees ten.

## Rebuilding the outline from the layout

A person reading the page finds the headings just by looking at it. Short text that is bigger or bolder than the text around it is a heading. Bigger means higher up. That is all layout-aware chunking does.

1. **Recover the outline.** Render the HTML page in a browser, or read the fonts out of the PDF. The most common text size is the body text. Short text that is larger or bolder than that is a heading. A page can use a dozen sizes, though, and making each one its own level gives you far too many. Group sizes that are close together into a few bins, at most six, the way HTML has H1 to H6. The biggest bin is the top level. Then hang every block of text under the nearest heading above it.
2. **Find the match.** Search the nodes of that tree for the question, by keywords or by embeddings.
3. **Expand along the tree.** Return the whole section the match belongs to, with its ancestor headings as a breadcrumb, such as `Refund Policy › Annual plans`.
4. **Deduplicate by node.** Keep track of which sections were already sent, so a follow-up question returns only new text.

![The four steps on the refund policy. Recover: font sizes become an outline of H1 and H2 headings with text under them. Find: the line about prorated refunds matches. Expand: the whole Annual plans section is returned, with the breadcrumb Refund Policy › Annual plans. Deduplicate: Annual plans is marked as sent, so a follow-up question gets only the How to request section](/images/layout-aware-chunking/5-approach.png)

There are two ways to do the search in step 2.

1. **Search only the leaf nodes.** Each leaf carries the breadcrumb of every heading above it, right up to the document title. A match on one line still tells the model which section it is in, and what that section is part of.
2. **Search any node.** When a node matches, return it with all of its children. This can return more text than the question needs. But if the match is on a section title, the whole section comes back with it, and that is often what answers the question.

The second one needs a limit, or a match on the document title returns the whole document. More on that below.

The first step is the only unusual part, and it is shorter than it sounds.

```python
def build_tree(blocks, body_size):
    root = Section(level=0, title="")
    open_sections = [root]
    for block in blocks:                      # in reading order
        if is_heading(block, body_size):      # short, and bigger or bolder than body text
            level = size_bin(block.size)      # one of up to six bins, biggest first
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

![Step 1 searches the whole document and matches the prorated refunds line, while the exception scores too low. The search climbs one level to "Annual plans". Step 2 searches again inside that section only, and the exception now ranks second and is kept](/images/layout-aware-chunking/3-climb-then-search.png)

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

This came out of a real failure, in an extraction system much like the [contract extraction agents](/when-one-agent-has-to-do-six-jobs/) I have written about before.

Picture a framework agreement with several supplier schedules back to back. Each schedule has its own payment terms, under a heading that is just bold text. We asked an agent for one supplier's payment terms. The chunk it got started halfway through the previous schedule, so it reported the wrong supplier's terms as the right one's. That is a lost boundary, and nothing in the chunk said so.

![Two supplier schedules back to back. A fixed chunk starts at Supplier A's payment terms and runs into Supplier B's heading, so the agent reports 60 days for Supplier B. Layout-aware chunking returns Supplier B's section only, and the agent reports 30 days](/images/layout-aware-chunking/4-contract-boundary.png)

Rebuilding the headings from the layout fixed most of it. In a test on PDFs, the share of reads that mixed up two records went from most of them to fewer than one in ten.

It also helps other agents that read documents. In that contract system, a discovery agent finds the parts of a contract that matter, and those parts rarely make sense on their own. In a [research assistant where agents call each other as tools](/agents-calling-agents-in-one-process/), a document worker reads whatever the user uploads. Both do better when the text they get comes with the headings that say what it is about.

## Reading only the part you need

There is another use, and it may be the more common one. An agent often needs one paragraph from a very long page.

Take the research assistant again. To answer a question about fuel costs, its web worker searches for what analysts are saying about whether fuel prices are rising or falling. One of the results is a policy document, fifty pages long. The assistant needs one paragraph of it. Nobody wants to send fifty pages to the model for that. It costs more, it is slower, and the paragraph that matters gets buried in everything else.

With the tree, the agent doesn't have to. It builds the tree for that one document, searches it by keyword and sends the model only the matching section, with its breadcrumb. The model gets one section instead of fifty pages, and it still knows which part of the document that section came from.

![A fifty-page policy document with sections on electricity, gas supply, fuel prices and transport. The agent builds the tree and searches it, and only the Analyst outlook section under Fuel prices goes to the model, with its breadcrumb](/images/layout-aware-chunking/6-fifty-pages.png)

## What it takes to make it work

It doesn't work out of the box. There are three things you will need to put effort into, test and tune. None of them is a dead end.

1. **Documents without a layout.** Plain text has no fonts or sizes to read. Numbering, blank lines and lines in capitals still show where sections start, and they can stand in for font sizes. Scanned PDFs need their page images read first.
2. **Rendering takes time.** Rendering an HTML page in a browser is slower than reading its text. Build the tree once for each document and keep it, so later questions don't pay for it again.
3. **Badly laid out documents.** If the author made body text bold for emphasis, it can look like a heading. Short and bold is a good starting rule. Test it on a sample of your own documents and adjust it until the outline matches what a person reading the page would see.

If your answers are often right in what they say but wrong about what it applies to, look at what your chunks lose before you look at your prompts.
