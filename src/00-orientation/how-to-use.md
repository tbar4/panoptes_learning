# How to Use This Course

## The two chapter types

Every arc alternates between two kinds of chapter, and they ask different things of you.

**Concept chapters** are the lecture. Read them without touching the keyboard. They end with a short list of questions you should be able to answer for yourself — if any are fuzzy, that is the signal to stop and ask before moving to the build. Do not proceed to a build chapter on a shaky concept; the whole point of frontloading concepts is that the build then feels obvious.

**Build chapters** are the pair-programming session. Here you write the code. Each one follows the same test-driven loop:

1. **Write the failing test.** You write it, from the behavior we described — not by copying an answer.
2. **Predict the failure.** Say out loud (or note down) what the compiler or test runner will report.
3. **Run it and check your prediction.** The gap between prediction and reality is the lesson.
4. **Write the minimal implementation.** Just enough to make the test pass. No more.
5. **Run again, watch it pass.**
6. **Commit.**

<div class="callout">
<span class="callout-label">New to Rust? Start with the Foundations arc</span>
Part I includes two concept chapters — Ownership and Moves, and Borrowing and References — before any real type-writing. These are read-only and have no build step, but they are the foundation the rest of the course assumes, especially the async arc. If Rust's ownership model is new to you, do not skip them; every later "why does the compiler want <code>&</code> here?" is answered there.
</div>

## The concept-check quizzes

Each part ends with a graded quiz. These are not busywork — they target the exact misconceptions that cause bugs three chapters later. Answer honestly before revealing; a wrong answer with a good explanation teaches more than a lucky guess. The score is for you, not for anyone else.

## The answer key

There is a companion implementation plan (linked in the appendix) that contains the full, worked version of every task. **Use it as an answer key, not a script.** Try each build yourself first. Check the plan after. The distance between your version and the plan's version is precisely where the learning lives — if they match, you understood it; if they differ, the difference is worth investigating.

## Working setup

You will want two things open side by side: this book, and a terminal in your project directory. A split screen or two monitors. The loop is fast — write test, run, read error, fix — and it only works if the feedback is immediate.

## When you get stuck

Bring the **exact** error message, verbatim. Rust's compiler errors are unusually good; most of the time the fix is in the error itself, and learning to read them is half of learning the language. When we work through a stuck point together, we dig into what the compiler is actually objecting to rather than papering over it.

<div class="callout trap">
<span class="callout-label">The one anti-pattern to avoid</span>
Copying a test or an implementation from the answer key <em>before</em> attempting it yourself. It feels productive and teaches almost nothing. Reproducing someone else's test does not build the instinct for writing your own, which is the actual skill.
</div>
