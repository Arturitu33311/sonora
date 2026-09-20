# Lyrics Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove the dead Apple Music lyrics source (the `binimum` client, confirmed 404 in production) and add a short retry to LrcLib so its occasional transient 503s don't drop a lyrics hit that a second try would have found.

**Architecture:** `binimum::Binimum` is one of several `LyricsProvider` implementations Sonora fans a lyrics query out to (`crates/sonora/src/main.rs`); removing it means deleting the module, every place that constructs it, and every place that names "Apple Music" as a lyrics-provider option, then confirming the workspace still compiles. LrcLib's fix stays local to its own `search()`: a small pure `retryable(status)` function decides whether a failure is worth retrying, and the request loop retries once after a short delay when it is.

**Tech Stack:** Rust, `reqwest`, `tokio::time::sleep` (the `time` feature is already enabled workspace-wide).

**Spec:** [docs/superpowers/specs/2026-09-20-spotify-first-crossfade-design.md](../specs/2026-09-20-spotify-first-crossfade-design.md), section "4. Arreglo real de las letras que fallan"

## Global Constraints

- Follow this repo's `CLAUDE.md`: Conventional Commits (`type(scope): description`, imperative, lowercase, no trailing period, no body), no `Co-Authored-By` trailer, ask before every `git push`.
- Do not touch `crates/music/src/apple/*` — that is the real Apple Music **streaming** provider (`AppleProvider`), unrelated to the `binimum` **lyrics** client despite both naming themselves "Apple Music".
- `cargo fmt`, `cargo check --workspace`, and `cargo clippy --workspace --all-targets` must stay clean after every task (per `CLAUDE.md`'s Checks section).

---

### Task 1: Remove the dead Apple Music (binimum) lyrics provider

**Files:**
- Delete: `crates/music/src/binimum/mod.rs` (and the now-empty `crates/music/src/binimum/` directory)
- Delete: `crates/music/src/lyrics/ttml.rs` (only `binimum` ever used it — confirmed by grep, zero other references)
- Modify: `crates/music/src/lib.rs:3`
- Modify: `crates/music/src/lyrics/mod.rs` (module declaration line)
- Modify: `crates/sonora/src/main.rs:106-113`
- Modify: `crates/state/src/settings.rs:354-360` (default list) and the `mod tests` block near the bottom
- Modify: `crates/views/src/screens/settings.rs:2549-2557`
- Modify: `crates/music/examples/lyrics_bench.rs`
- Modify: `crates/music/examples/lyrics_voices.rs`
- Modify: `crates/music/examples/lyrics-prober.rs`
- Modify: `assets/i18n/en-US/main.ftl`, `assets/i18n/uk/main.ftl`, `assets/i18n/pl/main.ftl`, `assets/i18n/ru/main.ftl` (each has one line: `settings-lyrics-provider-apple-music = Apple Music`)

**Interfaces:**
- Consumes: nothing new.
- Produces: nothing new — this task only removes a source, every caller of `LyricsProvider` and the `lyrics_providers` settings list keeps its existing shape (`Vec<Arc<dyn LyricsProvider>>`, `Vec<String>`).

- [ ] **Step 1: Write the failing test**

Add to the `mod tests` block in `crates/state/src/settings.rs` (near the other tests, using the same `use super::*;` already in scope):

```rust
#[test]
fn the_default_lyrics_providers_no_longer_include_a_dead_source() {
    let providers = Values::default().lyrics_providers;
    assert!(!providers.iter().any(|name| name == "Apple Music"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --package state the_default_lyrics_providers_no_longer_include_a_dead_source`
Expected: FAIL — the assertion fails because `"Apple Music"` is still in the default list.

- [ ] **Step 3: Delete the binimum module**

```bash
rm -rf crates/music/src/binimum
rm crates/music/src/lyrics/ttml.rs
```

- [ ] **Step 4: Remove the module declarations**

In `crates/music/src/lib.rs`, remove this line (currently line 3):

```rust
pub mod binimum;
```

In `crates/music/src/lyrics/mod.rs`, remove this line from the top module-declaration block:

```rust
pub(crate) mod ttml;
```

- [ ] **Step 5: Remove the provider registration in `main.rs`**

In `crates/sonora/src/main.rs`, the `lyrics` vector currently reads:

```rust
        let lyrics: Vec<Arc<dyn LyricsProvider>> = vec![
            Arc::new(music::spotify::SpotifyLyrics::from_env()),
            Arc::new(music::youtube::YouTubeLyrics::new()),
            Arc::new(music::binimum::Binimum::new()),
            Arc::new(music::musixmatch::Musixmatch::new()),
            Arc::new(music::lrclib::LrcLib::new()),
            Arc::new(music::kugou::Kugou::new()),
            Arc::new(music::netease::NetEase::new()),
        ];
```

Remove the `binimum` line so it reads:

```rust
        let lyrics: Vec<Arc<dyn LyricsProvider>> = vec![
            Arc::new(music::spotify::SpotifyLyrics::from_env()),
            Arc::new(music::youtube::YouTubeLyrics::new()),
            Arc::new(music::musixmatch::Musixmatch::new()),
            Arc::new(music::lrclib::LrcLib::new()),
            Arc::new(music::kugou::Kugou::new()),
            Arc::new(music::netease::NetEase::new()),
        ];
```

- [ ] **Step 6: Remove "Apple Music" from the default settings list**

In `crates/state/src/settings.rs`, the default reads:

```rust
            lyrics_providers: [
                "Spotify",
                "YouTube Music",
                "Apple Music",
                "Musixmatch",
                "LrcLib",
            ]
            .map(str::to_owned)
            .to_vec(),
```

Change it to:

```rust
            lyrics_providers: [
                "Spotify",
                "YouTube Music",
                "Musixmatch",
                "LrcLib",
            ]
            .map(str::to_owned)
            .to_vec(),
```

- [ ] **Step 7: Remove the Settings screen row entry**

In `crates/views/src/screens/settings.rs`, inside `lyrics_providers_row`, the `providers` array reads:

```rust
        let providers = [
            ("Spotify", "settings-lyrics-provider-spotify"),
            ("YouTube Music", "settings-lyrics-provider-youtube"),
            ("Apple Music", "settings-lyrics-provider-apple-music"),
            ("Musixmatch", "settings-lyrics-provider-musixmatch"),
            ("LrcLib", "settings-lyrics-provider-lrclib"),
            ("Kugou", "settings-lyrics-provider-kugou"),
            ("NetEase", "settings-lyrics-provider-netease"),
        ];
```

Remove the Apple Music line:

```rust
        let providers = [
            ("Spotify", "settings-lyrics-provider-spotify"),
            ("YouTube Music", "settings-lyrics-provider-youtube"),
            ("Musixmatch", "settings-lyrics-provider-musixmatch"),
            ("LrcLib", "settings-lyrics-provider-lrclib"),
            ("Kugou", "settings-lyrics-provider-kugou"),
            ("NetEase", "settings-lyrics-provider-netease"),
        ];
```

- [ ] **Step 8: Remove the unused i18n key**

Delete the line `settings-lyrics-provider-apple-music = Apple Music` from each of:
- `assets/i18n/en-US/main.ftl`
- `assets/i18n/uk/main.ftl`
- `assets/i18n/pl/main.ftl`
- `assets/i18n/ru/main.ftl`

- [ ] **Step 9: Fix the three example binaries that construct `Binimum`**

In `crates/music/examples/lyrics_bench.rs`:
- Remove `binimum` from the `use music::{ ... };` import list.
- Remove `"Apple Music",` from the `SOURCES` array.
- Remove `Arc::new(binimum::Binimum::new()),` from the `providers()` function.

In `crates/music/examples/lyrics_voices.rs`:
- Remove `binimum` from `use music::{Lyrics, LyricsProvider, LyricsQuery, Voice, binimum, musixmatch};` (leaving `use music::{Lyrics, LyricsProvider, LyricsQuery, Voice, musixmatch};`).
- Remove `Arc::new(binimum::Binimum::new()),` from the `providers` vec (leaving just the `musixmatch::Musixmatch::new()` entry).

In `crates/music/examples/lyrics-prober.rs`:
- Remove `binimum` from the `use music::{ ... };` import list.
- Remove `Arc::new(binimum::Binimum::new()),` from the `providers()` function.

- [ ] **Step 10: Run test to verify it passes**

Run: `cargo test --package state the_default_lyrics_providers_no_longer_include_a_dead_source`
Expected: PASS

- [ ] **Step 11: Confirm the whole workspace still compiles**

Run: `cargo check --workspace --examples`
Expected: no errors — this is the real safety net for this task, since Step 9 touches three files by hand and a missed reference anywhere fails the build.

- [ ] **Step 12: Format and lint**

Run: `cargo fmt && cargo clippy --workspace --all-targets`
Expected: no warnings.

- [ ] **Step 13: Commit**

```bash
git add -A
git commit -m "fix(music): drop the dead Apple Music lyrics source"
```

---

### Task 2: Retry LrcLib once on a transient server error

**Files:**
- Modify: `crates/music/src/lrclib/mod.rs`

**Interfaces:**
- Consumes: nothing new — this stays entirely inside `LrcLib::search`.
- Produces: `fn retryable(status: reqwest::StatusCode) -> bool` — a private free function later tasks do not need, but keep the name if you ever touch this file again from another task.

- [ ] **Step 1: Write the failing test**

Add to the `#[cfg(test)] mod tests` block at the bottom of `crates/music/src/lrclib/mod.rs` (it already has `use super::*;` in scope — check the existing tests there for the exact import line and match it):

```rust
#[test]
fn a_server_error_is_retryable_but_a_client_error_is_not() {
    assert!(retryable(reqwest::StatusCode::SERVICE_UNAVAILABLE));
    assert!(retryable(reqwest::StatusCode::INTERNAL_SERVER_ERROR));
    assert!(!retryable(reqwest::StatusCode::NOT_FOUND));
    assert!(!retryable(reqwest::StatusCode::OK));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test --package music a_server_error_is_retryable_but_a_client_error_is_not`
Expected: FAIL with "cannot find function `retryable`" — it does not exist yet.

- [ ] **Step 3: Write the minimal implementation**

Add these two constants near the top of `crates/music/src/lrclib/mod.rs`, beside the existing `SOURCE`/`ENDPOINT`/`AGENT` constants:

```rust
/// Total attempts against a transient failure: the first try plus one retry.
const ATTEMPTS: u8 = 2;
/// How long to wait before the retry. LrcLib is a small free service; a short pause is enough
/// to ride out the kind of blip that produced the 503 this exists for.
const RETRY_DELAY: std::time::Duration = std::time::Duration::from_millis(750);
```

Add this free function near the bottom of the file, beside `hit`/`filed`:

```rust
/// Whether a failed response is worth trying again. A 5xx is the server's own trouble, which a
/// short wait can outlast; a 4xx (a bad query, nothing found) will not change on a retry.
fn retryable(status: reqwest::StatusCode) -> bool {
    status.is_server_error()
}
```

Replace the current `search` body:

```rust
    async fn search(&self, query: &LyricsQuery) -> Result<Vec<LyricsHit>> {
        let response = self
            .http
            .get(ENDPOINT)
            .query(&[
                ("track_name", query.title.as_str()),
                ("artist_name", query.artist.as_str()),
            ])
            .header("User-Agent", AGENT)
            .send()
            .await
            .context("cannot reach lrclib")?;
        let status = response.status();
        if !status.is_success() {
            anyhow::bail!("lrclib answered with status {status}");
        }
        let found: Vec<Found> = response
            .json()
            .await
            .context("cannot read the lrclib response")?;
        Ok(found.into_iter().filter_map(hit).collect())
    }
```

with a version that retries once on a retryable status:

```rust
    async fn search(&self, query: &LyricsQuery) -> Result<Vec<LyricsHit>> {
        let params = [
            ("track_name", query.title.as_str()),
            ("artist_name", query.artist.as_str()),
        ];

        let mut attempt = 0u8;
        let response = loop {
            attempt += 1;
            let response = self
                .http
                .get(ENDPOINT)
                .query(&params)
                .header("User-Agent", AGENT)
                .send()
                .await
                .context("cannot reach lrclib")?;
            let status = response.status();
            if status.is_success() {
                break response;
            }
            if attempt >= ATTEMPTS || !retryable(status) {
                anyhow::bail!("lrclib answered with status {status}");
            }
            log::warn!("lyrics: lrclib answered with status {status}, retrying");
            tokio::time::sleep(RETRY_DELAY).await;
        };

        let found: Vec<Found> = response
            .json()
            .await
            .context("cannot read the lrclib response")?;
        Ok(found.into_iter().filter_map(hit).collect())
    }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cargo test --package music a_server_error_is_retryable_but_a_client_error_is_not`
Expected: PASS

- [ ] **Step 5: Run the whole crate's test suite**

Run: `cargo test --package music`
Expected: PASS — the existing `hit`/`filed` tests in this file are untouched by this change and must still pass.

- [ ] **Step 6: Format and lint**

Run: `cargo fmt && cargo clippy --workspace --all-targets`
Expected: no warnings.

- [ ] **Step 7: Commit**

```bash
git add crates/music/src/lrclib/mod.rs
git commit -m "fix(music): retry lrclib once on a transient server error"
```
