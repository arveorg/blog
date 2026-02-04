---
title: "MCP vs Proprietary (1 - 0)"
date: 2026-02-01
draft: false
tldr: "Hooked IDA Pro up as an MCP server and let AI agents reverse engineer a proprietary format for me."
---

I needed to support a proprietary file format. There was no public documentation, no open-source parser, and the vendor had no interest in changing that. The only path forward was to open up the binary and figure out what it does.

Which is how I ended up back in a disassembler for the first time in decades.

## Proprietary formats are a waste of everyone's time

I want to get this out of the way: proprietary file formats are almost never technically justified. They exist because someone decided that lock-in was more valuable than interoperability. The data inside is usually trivial. A header, some structured fields, maybe a compression layer. Nothing that couldn't be a well-documented open format. Nothing that benefits from being secret.

Every proprietary format creates a little ecosystem of people reverse engineering the same thing, independently, repeatedly, across years. All that collective effort could have gone toward building something useful instead. It's pure waste.

But here we are. The format exists, I need to read it, and nobody is going to hand me a spec. So let's talk about tools.

## Three disassemblers, three tradeoffs

I hadn't used IDA since Ilfak Guilfanov released it as shareware in the early 90s. I was running it on DOS. That's how long it had been. I'm not going to talk about what I was using it for, but iykyk ;) So I came into this round with fresh eyes, tried the three that people actually use for serious work, and came away with opinions.

[Ghidra](https://ghidra-sre.org/) is free, open source, and built by the NSA. Its real strengths are multi-binary project support, where you can load an application and all its libraries at once, and a built-in decompiler that covers every architecture it supports without charging extra. It's also fully extensible since you can read and modify the source. But the interface feels like it was designed by committee in 2005 and nobody has been allowed to touch it since. It works. It's not pleasant. Oh, yeah, it's using Java Swing..

[Binary Ninja](https://binary.ninja/) is the opposite experience. The UI is modern, responsive, and genuinely well-designed. Using it feels like someone who cares about user experience actually built a disassembler. You can orient yourself in an unfamiliar binary quickly, and the workflow just makes sense. For a lot of reverse engineering tasks, this is what I'd reach for first.

[IDA Pro](https://hex-rays.com/ida-pro) is the gold standard and it earns that reputation. The disassembler and decompiler produce output that is a noticeable step above the other two. The decompiled output is richer, the analysis is deeper, the type propagation is more accurate. For serious reverse engineering work, the quality of the output matters enormously, and IDA's output is *still* the best I've seen. The user experience, on the other hand, feels like it has accumulated thirty years of interface decisions without ever rethinking any of them. You learn to live with it because the results are worth it.

## Let the interns handle it

Here's where it gets interesting.

Staring at decompiled output of a proprietary format, manually tracing data structures, labeling fields, testing hypotheses about what each byte means. It's tedious, detail-oriented work. Exactly the kind of work I don't want to do by hand for hours on end.

So I hooked IDA Pro up as an [MCP server](https://github.com/mrexodia/ida-pro-mcp) (by the shockingly talented [Duncan Ogilvie](https://github.com/mrexodia)) and pointed my unreliable interns with amnesia at the problem, as he would have said.

If you've used LLMs for any kind of technical work, you know the type. They're enthusiastic, they work fast, they sometimes produce genuinely brilliant insights, and they forget everything between conversations. They'll confidently label a field as a checksum, then in the next session ask you what that same field is. Classic intern behavior.

But for reverse engineering a file format, this turns out to be a surprisingly good fit. The work is inherently exploratory. You make hypotheses, test them, refine them. Having an agent that can read disassembly, propose structure definitions, and iterate on them faster than I can type is genuinely useful. The amnesia is annoying but manageable. You keep notes. You feed context back in. You learn to work with the limitations.

The MCP integration means the agent can actually navigate IDA's analysis directly. It can look up cross-references, read decompiled functions, examine data segments. All the things I'd normally do by clicking around in the UI, except the agent does it programmatically and faster. My job shifts from doing the tedious work to directing the tedious work and verifying the results.

It's not perfect. The interns still need supervision. But they've gotten surprisingly reliable, and they get me to 80% in a fraction of the time. The remaining 20% is the interesting part anyway.

## The actual workflow

The loop looks like this: point the agent at a function that seems to handle file parsing, let it propose a data structure, validate that structure against known sample files, correct the mistakes, feed the corrections back in, and move to the next function. Repeat until you have a complete format specification.

What would have taken me days of manual analysis took an afternoon of supervised agent work. The proprietary format is no longer proprietary to me.

And honestly, that felt really good.
