# Concept: Testing Against a Mock Server

> **Kind:** Concept.

## The idea

You need to test that your Anthropic client correctly builds a request and parses the response. You do **not** want those tests to hit the real API — that costs money, needs a key, is slow, and is non-deterministic. The solution is a **mock server**: a fake HTTP server, started inside the test, that returns a canned response you control.

`wiremock` spins up a real local HTTP server on a random port, lets you say "when a POST arrives at `/v1/messages`, respond with this JSON," and gives you its URL. You point your client at that URL instead of the real API. The client cannot tell the difference — it makes a genuine HTTP request; the server just happens to be under your control.

## Why this is the right design pressure

To make this work, your client must accept its base URL as a parameter rather than hard-coding `https://api.anthropic.com`. That injectability is good design independent of testing — it is also how you would point at a proxy or a rented-inference gateway. The test forces a seam that turns out to be useful in production. Testable code and flexible code are frequently the same code.

<div class="callout">
<span class="callout-label">The payoff</span>
Every request-building and response-parsing bug is caught with zero API spend and full determinism. You can run these tests a thousand times in CI and never pay a cent or flake on a network hiccup.
</div>

## Questions to lock

1. Why is hitting the real API in a unit test a bad idea, on at least three counts?
2. What does `wiremock` actually start, and why can the client not tell it is talking to a mock?
3. Why does mock-testing *force* the client to take its base URL as a parameter, and why is that good beyond testing?

{{#quiz ../../quizzes/03-async-concept-mocking.toml}}
