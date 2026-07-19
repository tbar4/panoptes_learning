# Concept: Testing Against a Mock Server

> **Kind:** Concept. **New crate:** `wiremock` — this chapter shows it working before you build with it.

## The idea

You need to test that your Anthropic client correctly builds a request and parses the response. You do **not** want those tests to hit the real API — that costs money, needs a key, is slow, and is non-deterministic. The solution is a **mock server**: a fake HTTP server, started inside the test, that returns a canned response you control.

The crucial thing to internalize — and the first quiz question — is that `wiremock` is not a mocked *object* or a stubbed *function*. It starts a **real HTTP server** on a random local port. Your client makes a genuine network request to it; the server just happens to be under your control. The client cannot tell the difference, so the entire request-building and response-parsing path is exercised for real.

> These examples use `wiremock`, which is not available on the Rust playground, so there is no play button. To run them, make a scratch crate with `anyhow`, `reqwest` (features `["json"]`), `serde`/`serde_json`, `tokio` (features `["full"]`) as dependencies and `wiremock = "0.6"` under `[dev-dependencies]` — the same set the harness crate declares.

## A mock server is a real server

Start one, teach it one behavior, and hit it with an ordinary HTTP client:

```rust,ignore
use wiremock::matchers::{method, path};
use wiremock::{Mock, MockServer, ResponseTemplate};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. A real HTTP server starts on a random local port.
    let server = MockServer::start().await;
    println!("mock listening at {}", server.uri());

    // 2. Teach it exactly one behavior.
    Mock::given(method("POST")).and(path("/ping"))
        .respond_with(ResponseTemplate::new(200).set_body_json(serde_json::json!({ "ok": true })))
        .mount(&server).await;

    // 3. Any HTTP client can hit it — it is a socket, not a fake object.
    let body: serde_json::Value = reqwest::Client::new()
        .post(format!("{}/ping", server.uri()))
        .send().await?
        .json().await?;
    println!("matched:   {body}");

    // 4. A request no mock matches gets 404 — silently.
    let miss = reqwest::Client::new()
        .post(format!("{}/pong", server.uri()))
        .send().await?;
    println!("unmatched: {}", miss.status());
    Ok(())
}
```

```text
mock listening at http://127.0.0.1:52866
matched:   {"ok":true}
unmatched: 404 Not Found
```

The vocabulary, line by line:

- `MockServer::start().await` — binds a fresh server to a random free port. Every test gets its own; tests can run in parallel without colliding.
- `Mock::given(matcher).and(matcher)…` — a chain of **matchers** describing which incoming requests this mock applies to: HTTP method, path, headers, even body contents.
- `.respond_with(ResponseTemplate::new(200).set_body_json(…))` — what to send back when the matchers match: status code plus a canned JSON body you write out by hand.
- `.mount(&server).await` — registers the mock on the server. Until mounted, it does nothing.
- `server.uri()` — the mock's base URL (`http://127.0.0.1:<port>`). This is the value you hand to your client instead of the real API's URL.

And note line 4: a request that matches *no* mounted mock is answered with a plain `404`. Keep that in mind — it is the classic wiremock trap, and we will come back to it.

## `#[tokio::test]`

HTTP is async, so these tests are `async fn` — but the plain `#[test]` attribute cannot run an async function. `#[tokio::test]` replaces it: it spins up a tokio runtime for that one test and runs your async body to completion, exactly like `#[tokio::main]` does for a binary. This is why the harness crate's manifest needs `tokio` even though the dispatch binary is the only "real" async entry point.

## The full pattern: a client with an injectable base URL

Here is the complete shape the build chapter asks of you, on a toy service so it cannot be mistaken for the answer key: a fortune-cookie API. `POST {base}/v1/fortunes` with an `x-api-key` header and a JSON body; the response nests the payload two levels deep, and the client parses it into a clean public type:

```rust,ignore
use serde::Deserialize;

pub struct FortuneClient {
    pub api_key: String,
    pub base_url: String, // injectable so tests can point at a mock
    http: reqwest::Client,
}

impl FortuneClient {
    pub fn new(api_key: String, base_url: String) -> Self {
        Self { api_key, base_url, http: reqwest::Client::new() }
    }

    pub async fn tell(&self, topic: &str) -> anyhow::Result<Fortune> {
        let raw: RawResp = self.http
            .post(format!("{}/v1/fortunes", self.base_url))
            .header("x-api-key", &self.api_key)
            .json(&serde_json::json!({ "topic": topic }))
            .send().await?
            .error_for_status()?
            .json().await?;
        Ok(Fortune { text: raw.fortune.text, credits_used: raw.credits.used })
    }
}

/// What callers get: flat, typed, ours.
#[derive(Debug, PartialEq)]
pub struct Fortune {
    pub text: String,
    pub credits_used: u32,
}

/// What the wire carries: nested, shaped by someone else's API.
#[derive(Deserialize)]
struct RawFortune { text: String }
#[derive(Deserialize)]
struct RawCredits { used: u32 } // "remaining" exists in the JSON; serde ignores undeclared fields
#[derive(Deserialize)]
struct RawResp { fortune: RawFortune, credits: RawCredits }
```

And the test — a mock that both *responds* and *asserts*:

```rust,ignore
#[cfg(test)]
mod tests {
    use super::*;
    use wiremock::matchers::{header, method, path};
    use wiremock::{Mock, MockServer, ResponseTemplate};

    #[tokio::test]
    async fn parses_a_mocked_fortune() {
        let server = MockServer::start().await;
        Mock::given(method("POST"))
            .and(path("/v1/fortunes"))
            .and(header("x-api-key", "test-key"))
            .respond_with(ResponseTemplate::new(200).set_body_json(serde_json::json!({
                "fortune": { "text": "You will refactor fearlessly." },
                "credits": { "used": 1, "remaining": 41 }
            })))
            .expect(1)
            .mount(&server)
            .await;

        let client = FortuneClient::new("test-key".into(), server.uri());
        let f = client.tell("rust").await.unwrap();
        assert_eq!(f.text, "You will refactor fearlessly.");
        assert_eq!(f.credits_used, 1);
    }
}
```

This runs green — one real HTTP round trip, zero network, zero spend. The mapping onto your build is one-for-one: `FortuneClient` ↔ `AnthropicClient`, `/v1/fortunes` ↔ `/v1/messages`, the `x-api-key` header appears in both, `tell()` ↔ `generate()`, the `Raw*` structs ↔ `RawResp`/`RawBlock`/`RawUsage`, and `Fortune` ↔ `ModelResponse`. The domain is different; the shape is identical.

Notice what the matcher chain is doing in the test: `header("x-api-key", "test-key")` means the mock only matches if your client *actually sent* that header. The mock is not just a canned response — it is also an assertion about the request your code built. If you forget the header in `tell()`, this test fails.

<div class="callout trap">
<span class="callout-label">The silent-404 trap, and `.expect(1)`</span>
When your matcher chain is wrong — a typo in the path, a missing header — the mock never matches, and wiremock answers <code>404</code>. Your client then fails with a confusing "404 Not Found" that looks like a client bug rather than a test-setup bug. <code>.expect(1)</code> is the guard: it declares this mock must be matched exactly once, and when the server shuts down at the end of the test, wiremock <em>verifies</em> that and fails loudly, naming the mock that went unmatched. Cheap insurance; use it.
</div>

One more detail worth noticing: `RawCredits` declares only `used`, while the JSON also carries `remaining`. serde ignores fields you did not declare — which is exactly why your `RawResp` for the real API can model just `content` and `usage` and skip the dozen other fields the response carries.

## Why this is the right design pressure

To make any of this work, your client must accept its base URL as a parameter rather than hard-coding `https://api.anthropic.com` — that is the `base_url` field and the `server.uri()` handoff above. That injectability is good design independent of testing: it is also how you would point at a proxy or a rented-inference gateway. The test forces a seam that turns out to be useful in production. Testable code and flexible code are frequently the same code.

<div class="callout">
<span class="callout-label">The payoff</span>
Every request-building and response-parsing bug is caught with zero API spend and full determinism. You can run these tests a thousand times in CI and never pay a cent or flake on a network hiccup.
</div>

## Questions to lock

1. Why is hitting the real API in a unit test a bad idea, on at least three counts?
2. What does `wiremock` actually start, and why can the client not tell it is talking to a mock?
3. In the fortune test, what happens if `tell()` forgets to set the `x-api-key` header — which line of the test catches it, and what status does the client see?
4. What does `.expect(1)` verify, and when does that verification run?
5. Why does mock-testing *force* the client to take its base URL as a parameter, and why is that good beyond testing?

{{#quiz ../../quizzes/03-async-concept-mocking.toml}}
