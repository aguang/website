---
title: What's new January and a bit of February 2025
draft: false
categories: ["posts", "R", "phylogenetics", "VSCode"]
tags: updates
classes: wide
---

I didn't make this a 2025 resolution or anything, but after writing the quick SRA post last month and reviving my site I realized it doesn't take all that much effort for me to put things together as I'm basically writing notes all the time anyway to document. Why not a summary document of my week/month?

I tackled Advent of Code again last December, which I first tried in 2022 and then skipped in 2023. I didn't get very far, only to Day 5, but used it as an opportunity to learn Rust which did take a good amount of time. I took tons of notes in each day's READMEs. I think when it rolls around again in December I may either try R and time myself for how long it takes me to actually program each day's solutions as well as hopefully get further than Day 7 with my daily driver language. Or, maybe I'll work on my Rust proficiency more with a goal of getting further than Day 5 and also timing myself in terms of how long each solution takes me to have a reference for future years. This will probably be more useful to me than benchmarking my code runtimes, although benchmarking is always a fun exercise.

Work is kicking off quickly with various grant submissions that I am helping with. And I hope to publish a few first-author papers soon.

Some R-related irritations I seem to keep needing to remind myself of:

 * The root node of an `ape` phylogenetic tree object `phylo` is `Ntip(phylo)+1`.
 * To get the LaTeX code out of a `kable` table, it seems the only way to do so is to actually render the document into a pdf and have the [`keep_tex: true`](https://stackoverflow.com/questions/57093999/let-knitr-kable-display-latex-code-for-further-editing) option. However, I am now using Quarto and these instructions while similar are not actually [the same for Quarto](https://quarto.org/docs/output-formats/pdf-basics.html#latex-output). It turns out you have to do:

```
format:
  pdf:
    documentclass: report
    keep-tex: true
```

WHY CAN I NOT JUST GET THE LATEX OUTPUT DIRECTLY IN THE CONSOLE?

Some other irritations I also keep needing reminders of:

 * [Turning on and off wordwrap](https://stackoverflow.com/questions/31025502/how-can-i-switch-word-wrap-on-and-off-in-visual-studio-code) with VSCode is ⌥+Z