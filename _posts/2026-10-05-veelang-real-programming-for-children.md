---
layout: post
title: VeeLang - real programming, in words children already know
excerpt: For a while now I have been building a small programming language for children, called VeeLang. Children type plain English, it draws pictures, and along the way they meet the same ideas programmers use every day. This post is about why I built it and how it teaches.
tweet:
published: true
---

For a while now I have been building a small programming language for children, called [VeeLang](https://veelang.org). It is live now, it is free, and I would love to hear what you think of it. This post is about why I built it and how it tries to teach.

<video controls muted playsinline preload="metadata" poster="/images/veelang/veelang-demo-poster.jpg" style="width: 100%; border-radius: 8px;">
  <source src="/images/veelang/veelang-demo.mp4" type="video/mp4">
  Your browser cannot play this video. You can <a href="/images/veelang/veelang-demo.mp4">download it</a> instead.
</video>

## Why another language for children?

I believe children can be introduced to real programming concepts, and eventually quite complex ones, much earlier than we usually try. What gets in the way is not the concepts. It is the language they meet them in. Curly braces, semicolons and words like `def` and `var` mean nothing to a seven-year-old, so the first weeks of learning go into decoding symbols instead of thinking.

Children already have three things going for them:

- They speak plain English
- They love drawing and colouring
- They understand shapes

VeeLang puts those three together. A child types

```
draw circle at 120, 80 having radius 50, fill "gold"
```

and a gold circle appears. Every line reads like a sentence, and every sentence draws something, so a child can see straight away what their code did.

## One idea at a time

The drawing is the hook, but the point is the ideas underneath. VeeLang introduces them one at a time, and each one is something a child can see.

**Variables and maths.** Give a number a name, then use it again.

```
number big = 50
draw circle at 70, 80 having radius big / 2, fill "orchid"
draw circle at 160, 80 having radius big, fill "plum"
```

**Objects and properties.** Every shape has properties, and you can change them.

```
circle ball at 70, 80 having radius 40, fill "skyblue"
ball.fill = "tomato"
ball.x = 150
draw ball
```

**Loops.** Repeat something without writing it again. Each time round, `count` goes up by one.

![A loop in VeeLang and the five circles it draws](/images/veelang/loop.jpg)

**Functions.** In VeeLang they are called recipes: write a shape once, give it settings, then use it as many times as you like.

```
shape face having size 60 {
  circle having radius size / 2, fill "gold"
  circle at -size / 5, -size / 8 having radius size / 12, fill "black"
  circle at size / 5, -size / 8 having radius size / 12, fill "black"
}
draw face at 60, 80
draw face at 160, 80 having size 110
```

**Conditions.** A recipe can change what it draws depending on a setting, such as a cloud that shows a sun when the weather is sunny and rain when it is rainy.

None of these are toy versions. They are the same ideas a professional programmer uses every day, introduced in words a child can read. When a child later moves on to Python or JavaScript, the ideas will already be familiar. Only the spelling changes.

## How far can it go?

Put those ideas together and the pictures get much richer. This village is about 100 lines of VeeLang. There are recipes for the houses, trees, boats and birds; loops for the windows, the fence and the sun's reflection on the lake; and a `when` that turns the lights on or off in each house.

![A village by a lake at sunset, drawn with VeeLang](/images/veelang/village.png)

This is the house recipe. Every house in the picture comes from it, each with its own settings:

```
shape house having floors 1, wall "wheat", roof "firebrick", lights "on" {
  rectangle at 0, -50 * floors having width 100, length 50 * floors, color wall
  rectangle at 70, -50 * floors - 30 having width 14, length 30, color "dimgray"
  triangle having corner1 at -12, -50 * floors, corner2 at 112, -50 * floors,
    corner3 at 50, -50 * floors - 42, color roof
  rectangle at 40, -32 having width 20, length 32, color "saddlebrown"
  when lights is "on" {
    repeat floors times with f {
      square at 12, -50 * f + 12 having size 22, color "gold"
      square at 66, -50 * f + 12 having size 22, color "gold"
    }
  }
  when lights is "off" {
    repeat floors times with f {
      square at 12, -50 * f + 12 having size 22, color "lightsteelblue"
      square at 66, -50 * f + 12 having size 22, color "lightsteelblue"
    }
  }
}
```

The whole program is short enough to read in one sitting. [Open the village in the playground](https://play.veelang.org/#code=ztVfLbus2EN37KwbaFEjkWm873V3cdVGg6A_QEmURpkmXpGKkRf69GJKSKNnRTS7QhQF5RJ55nXlot4Nv8Mo4JycKxzcwHQVOzjQGYkD3QlOz-U7EK9HjqY68MnGCG2tMB4ckiYFTcTIdlPjcMs4h4uzUGU34RYpos9nt4K-Ogj6_wYkaDQ1RZ6rAyBtRjbY6jbxuGkVuoGhtiDhxigYkMSQf6zskMdSSSwWR5sTQI-9p9BjlsAKzn2AutGH95dqrK_8IKS0_ByVV3bHmA5AsWwGpJhAbxloqwoMo9iIGzcQZbx9px0RjI3iRvTCECe1U1kzVTl-JgNkUAUUa1ms4HKbo-USFF3UvVi4Hjp4kb5x1vw8WQEsUkBt5c4hGMee8x6ilElSlCL9F-Dy3aCjMUJi-YJyrUZijsCimk1YvkmiZ9hVN9upSU4n_s6ScadpXC01f0FIVD7RYwCyf-3M4rPuDAf2T1uxK9W9A4MhUEwOBmsvePhhFsUihk72mQESDhyQxG92RK7XnB_s0-4dCmsG_GwCiamiVvLjSMtK-DKrM59dJh6pgjUAiWsNiMB2rz4JqDVkIOAB5THiC7Odx3zfeD-vvzJEqsY7IV8LnNTRp9YVkBTsUeIVXJs7RcBfZ50_k8fhY3WP-CG-0FVMyXO8oejYYO6v_bRHD1r_fQb5oBKO26URQp03D6VHJm4gs6pUis8GwC9VwY6YDY_XBas0Fup-hhCcwoUEH2E663VuLCCGlA4jt1yFy3wanG9mv2f2dsDJOilLr9PsUb0d972DLpVQa0hhuBCfQraPERDEoKVuIWqboUbH6HMXgZhNEUkT3yUGrygSeBrxZctLZsBsPDYai4iXefgm4hXzR-9NiBA3bAbucFHlDl9dymWZz_HkjvXu7WSShvDevGPmNoVv6g_1tm2dzD7IpLHn2EVtvHRVD7FkQ_pHH3oKAzK0_AKD_7omyBgQuwTN2tbA3ZNliLC1vV9VXb79v3O_O_Lb9X-13qgyl3M2Dn_XkEc77vI5wbIzXCbPFwwx17q2QD0fIjG8vocTyKy1jSA8fUWIFe7q4gr5fRedMUDeaitIPJnzY7stJn7vHqBBkNoHydeuKA-LMnbeyaj7jy0GEejC0GPTdDr7jVNN2auOg1sCE3eH02a9MbuzZOCQx7JOFtNhb6SznRbnxRCw8BzF99p5dBoiBrEzgGQokTY27WoyrFmwhLQbJfGUovLm4dp4U0c7i4RPh8Wqb55_cjzl7pY0ix7sVGbE9WJGsgNluPKRwIniwJ_-iQdGW09owKdynh74QzqmCtlemo35L9XErl3ELGPSSuCobI4emPUN6GCNnJFS5jebKKVt-c_bZVhNyL4y6_-Ya9k5qA2M32Wkl91PzxZPEL4TGfkcEx3x7yhanUiRpcMxNT9s17HozTFBDCQ-HZ9tGC6vSh2btl2blefnIrjwOVU_5HPTb1dhNxJnaonqoNk3vwvGSPHD03JEzm5S4TjDXUJXlIw2HxGbpG7RU1BQruFVSGJCtrZBZ5iYioQ37DMliqYzPczoce8XfblI2i37kOZpVH5M0T0cybm0DzSur6V7-Un5K68DEP3oD0nWoGzFUOafs2LATDSu1TOZS-yVUBD3KDhab3DfKubxFm_8A) to read it, change it and see what happens. Try turning the lights off in every house, or adding a floor.

## Mistakes are part of learning

Children make mistakes, and that is where a lot of the learning happens. So VeeLang never shows a cryptic error. If a child types `draw cirle`, it asks whether they meant `circle`. If a colour is misspelt, it suggests the closest one. Every message is written for a child, and every one comes with a suggestion for what to try next.

![The VeeLang playground helping with a typo: line 7 says draw cirle, and the help panel says "I don't know 'cirle'. Did you mean 'circle'?"](/images/veelang/mistake.jpg)

## Safe to hand to a child

I wanted parents and teachers to be able to hand VeeLang to a child without having to think twice:

- It is free, with no accounts, no sign-up and no ads.
- Drawings are saved in the browser, on the child's own device. They are never uploaded anywhere.
- Sharing a drawing puts it inside the link itself, so nothing is stored on a server.
- There are step-by-step lessons, and the playground works in any modern browser.

## Try it, and tell me what you think

It is still early, and I would really value other people's eyes on it:

- If you are a **parent of young children**, try it together and tell me what they made, and where they got stuck.
- If you **work in education**, I would love to hear whether this could work in a classroom, and what it would need.
- If you are simply **interested in how children learn to code**, I would be glad to connect and swap ideas.

You can find everything at [veelang.org](https://veelang.org), start drawing straight away in the [playground](https://play.veelang.org), or follow the [lessons](https://learn.veelang.org). Feedback, a conversation or a collaboration are all welcome. The easiest way to reach me is [@suhas_chatekar on X](https://x.com/suhas_chatekar).
