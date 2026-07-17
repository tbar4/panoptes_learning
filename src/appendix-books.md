# Appendix: Reference Books

This course is grounded in five books. Where a chapter anchors a concept to a specific book, it cites it inline. This appendix says what each one is for, so you know where to go deeper.

## Async Rust — Maxwell Flitton & Caroline Morton (O'Reilly)

The primary source for **Part IV**. Its treatment of the future lifecycle (idle → polled → Pending/Ready), the runtime's polling loop, and the concurrency model underpins the async concept chapters. If any async idea in this course feels thin, this is the book to open — its chapter on futures, pinning, and context goes deeper than a working harness strictly needs, which is exactly why it is the right reference when the compiler surprises you.

## Command-Line Rust — Ken Youens-Clark (O'Reilly)

The source for **Part VII**. Its early chapters model exactly what the CLI arc does: building tools with `clap`, writing integration tests that run the compiled binary, and — the idea that matters most — exit codes and composability ("Exit Values Make Programs Composable"). The book's discipline around honest exit status is what turns `panoptes` from three scripts into a composable tool.

## Rust for Rustaceans — Jon Gjengset (No Starch)

The source for the **Foundations arc** (ownership, moves, borrowing) and a best-practices reference throughout. Its Chapter 1 "Foundations" gives the flows mental model for ownership and lifetimes that the course teaches, and its intermediate-idioms chapters inform trait and API design decisions. This is the book to grow into after the course.

## Effective Rust — David Drysdale (O'Reilly)

A best-practices reference, organized as discrete numbered "items" (in the tradition of *Effective C++*). Consulted for idiomatic choices around the type system, error handling, and API design. Where a design decision in the harness has an idiomatic "right answer," this book usually has an item on it.

## AI Engineering — Chip Huyen (O'Reilly)

Foundational knowledge for the **evaluation methodology** the harness serves, and the broader thread connecting Panoptes to the thesis and to Norion. It sharpens the wrap-up's framing of what a capability benchmark is and why the codebook, reliability, and analysis stages are shaped the way they are. Read it alongside the eval-engineering canon (Husain, Yan, Shankar) for the discipline this harness is an instance of.

<div class="callout">
<span class="callout-label">On quotation and copyright</span>
This course quotes these books only in short, attributed fragments to anchor precise definitions, and otherwise teaches the concepts in its own words. The books are the authority; the lessons are the application. For the full treatment of any concept, go to the source — the citations point you to the right chapter.
</div>
