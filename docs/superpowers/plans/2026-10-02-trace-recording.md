# Trace Recording Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add `--pipe` mode with `--record` to Selenium Manager (Rust) so all bindings get trace recording via HTTP proxy + BiDi screenshots, producing Playwright-compatible trace.zip.

**Architecture:** Selenium Manager in `--pipe` mode starts a local HTTP proxy, forwards binding requests to WebDriver, captures screenshots via BiDi WebSocket after each action, and packages trace.zip on session close.

**Tech Stack:** Rust (Selenium Manager), tower-http (HTTP proxy), tokio-tungstenite (BiDi WebSocket client), tokio (async runtime), reqwest (HTTP client), zip (archiver), serde_json (trace.json serialization)

---

### Task 1: Add HTTP server + WebSocket dependencies

**Files:**
- Modify: `rust/Cargo.toml`
- Modify: `rust/BUILD.bazel`

- [ ] **Step 1: Add tower-http and tokio-tungstenite to Cargo.toml**

```toml
# In [dependencies] section, after tokio line:
tokio-tungstenite = { version = "0.27", default-features = false, features = ["client"] }
tower-http = { version = "0.6", default-features = false, features = [] }
```

These versions match what's already in the crate dependency tree.

- [ ] **Step 2: Add tower-http + tokio-tungstenite to BUILD.bazel crate deps**

In `rust/BUILD.bazel`, `rust_library` and `rust_binary` rules use `all_crate_deps(normal = True)` which auto-discovers deps from Cargo.toml. No changes needed for Bazel.

- [ ] **Step 3: Verify compilation**

```bash
cargo build 2>&1 | tail -5
# Expected: new deps compile, no errors
```

---

### Task 2: Create pipe module — HTTP proxy + BiDi bridge

**Files:**
- Create: `rust/src/pipe.rs`
- Modify: `rust/src/lib.rs` (add `pub mod pipe;`)

- [ ] **Step 1: Write failing test**

```rust
// tests/pipe_tests.rs or inline in pipe.rs
// For now: verify the module compiles and exports PipeMode struct
```

- [ ] **Step 2: Create pipe.rs with proxy server**

```rust
use anyhow::Error;
use std::net::SocketAddr;
use std::path::PathBuf;
use tower::Request;
use tower::Response;
use tower_http::Router;
use tower_service::Serve;

pub struct PipeConfig {
    pub webdriver_url: String,
    pub webdriver_ws_url: Option<String>,
    pub record_path: Option<PathBuf>,
}

pub struct PipeMode {
    config: PipeConfig,
    proxy_port: u16,
    recorder: Option<Recorder>,
}

impl PipeMode {
    pub fn new(config: PipeConfig) -> Self {
        Self { config, proxy_port: 0, recorder: None }
    }

    /// Start HTTP proxy server on a random port.
    /// Returns the port number the server is listening on.
    pub fn start(&mut self) -> Result<u16, Error> {
        let app = Router::new()
            .route("{*path}", |req: Request| self.handle_request(req));

        let port = 0; // let OS assign
        let mut serve = Serve::new(app);
        let addr = SocketAddr::from("0.0.0.0:0");
        let listener = serve.bind(&addr)?;
        self.proxy_port = listener.local_addr().port();

        // Spawn server in background thread
        thread::spawn(|| serve.run());

        Ok(self.proxy_port)
    }

    fn handle_request(&self, req: Request) -> Result<Response, Error> {
        // Forward to WebDriver
        let url = format!("{}{}", self.config.webdriver_url, req.url().path());
        let body = req.body().to_string();

        let response = reqwest::Client::new()
            .request(req.method().to_string(), url)
            .headers(req.headers().clone())
            .body(body)
            .send()?;

        // Rebuild tower-http Response from reqwest Response
        Ok(Response::new(
            response.status(),
            response.headers().clone(),
            response.body().to_string(),
        ))
    }
}
```

- [ ] **Step 3: Add module export in lib.rs**

```rust
// In lib.rs, after existing pub mod lines:
pub mod pipe;
```

- [ ] **Step 4: Write test that starts pipe and verifies it binds a port**

```rust
// In pipe.rs or tests/
fn test_pipe_starts_proxy() {
    let config = PipeConfig {
        webdriver_url: "http://localhost:1",
        webdriver_ws_url: None,
        record_path: None,
    };
    let mut pipe = PipeMode::new(config);
    let port = pipe.start()?;
    assert(port > 0);
}
```

- [ ] **Step 5: Compile and run test**

```bash
cargo test --test pipe_tests -v 2>&1 | Select-String "PASS"
# Expected: test passes
```

---

### Task 3: Implement WebDriver lifecycle management

**Files:**
- Modify: `rust/src/pipe.rs`

- [ ] **Step 1: Add WebDriver process management**

```rust
// In pipe.rs, add:
use std::process::{Child, Command};
use std::path::Path;

impl PipeMode {
    pub fn start_webdriver(driver_path: &Path, port: u16) -> Result<Child, Error> {
        let cmd = Command::new(driver_path.to_string_lossy())
            .arg(format!("--port={}", port))
            .stdout(process::Stdio::null())
            .stderr(process::Stdio::null())
            .spawn()?;
        Ok(cmd)
    }
}
```

- [ ] **Step 2: Add session ID extraction from POST /session response**

```rust
// In PipeMode::handle_request, after forwarding POST /session:
fn extract_session_id(response_body: &str) -> Option<String> {
    // Parse JSON: {"value": {"sessionId": "...", "capabilities": {"webSocketUrl": "..."}}}
    let parsed = serde_json::from_str::<serde_json::Value>(response_body)?;
    parsed["value"]["sessionId"].as_str().cloned()
}
```

- [ ] **Step 3: Test WebDriver forwarding logic (mock)**

Integration test with actual WebDriver will come in Task 7.

---

### Task 4: Create recorder module — screenshot capture via BiDi

**Files:**
- Create: `rust/src/recorder.rs`
- Modify: `rust/src/lib.rs` (add `pub mod recorder;`)

- [ ] **Step 1: Define Recorder data structures**

```rust
use std::path::PathBuf;
use std::time::{SystemTime, Duration};

pub struct RecordedAction {
    pub command: String,         // e.g. "click", "navigate"
    pub url: String,
    pub method: String,          // HTTP method
    pub request_body: String,
    pub response_body: String,
    pub status: u16,
    pub timestamp: u64,          // unix ms
    pub duration_ms: u64,
    pub screenshot_bytes: Option<Vec<u8>>,
}

pub struct Recording {
    pub path: PathBuf,           // output trace.zip path
    pub actions: Vec<RecordedAction>,
    pub session_id: Option<String>,
    pub start_time: u64,
}
```

- [ ] **Step 2: Implement BiDi screenshot capture**

```rust
use tokio::net::TcpStream;
use tokio_tungstenite::Client as WsClient;

pub struct BiDiScreenshotter {
    ws: WsClient<TcpStream>,
    next_id: u64,
}

impl BiDiScreenshotter {
    pub fn connect(ws_url: &str) -> Result<Self, Error> {
        let ws = WsClient::connect(ws_url)?;
        Ok(Self { ws, next_id: 1 })
    }

    pub fn take_screenshot(&mut self) -> Result<Vec<u8>, Error> {
        let id = self.next_id;
        self.next_id += 1;

        let cmd = serde_json::to_string(Map {
            "id": id,
            "method": "browserContext.takeScreenshot",
            "params": Map {},
        })?;

        self.ws.send(cmd)?;

        // Read response (blocking read, or use async channel)
        let response = self.ws.recv()?;
        let parsed = serde_json::from_str::<serde_json::Value>(response)?;

        // Response: {"id": id, "result": {"data": "<base64>"}}
        let base64_data = parsed["result"]["data"].as_str()?;
        let decoded = base64::decode(base64_data)?;
        Ok(decoded)
    }
}
```

- [ ] **Step 3: Wire screenshot capture into proxy flow**

```rust
// In PipeMode::handle_request:
fn should_screenshot(method: &str, path: &str) -> bool {
    let action_paths = vec![
        "/url", "/elements", "/click", "/actions",
        "/back", "/forward", "/refresh", "/keys",
    ];
    action_paths.any(|p| path.contains(p))
}
```

- [ ] **Step 4: Write tests for should_screenshot logic**

```rust
fn test_should_screenshot_for_action() {
    assert(should_screenshot("POST", "/session/1/url"));
    assert(should_screenshot("POST", "/session/1/elements"));
    assert(!should_screenshot("GET", "/session/1/title"));
    assert(!should_screenshot("GET", "/session/1/cookies"));
}
```

---

### Task 5: Create archiver module — trace.zip packaging

**Files:**
- Create: `rust/src/archiver.rs`
- Modify: `rust/src/lib.rs` (add `pub mod archiver;`)

- [ ] **Step 1: Define trace.json schema**

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize)]
pub struct Trace {
    pub r#type: String,  // "trace"
    pub version: u32,
    pub pages: Vec<TracePage>,
}

#[derive(Serialize)]
pub struct TracePage {
    pub page_id: String,
    pub title: String,
    pub url: String,
    pub commands: Vec<TraceCommand>,
}

#[derive(Serialize)]
pub struct TraceCommand {
    pub r#type: String,
    pub action: String,
    pub timestamp: u64,
    pub duration: u64,
    pub screenshot_index: Option<u32>,
    pub url: String,
    pub response: serde_json::Value,
}
```

- [ ] **Step 2: Implement package() that writes trace.zip**

```rust
use std::fs::File;
use std::path::Path;
use zip::ZipWriter;

pub fn package(recording: &Recording, output: &Path) -> Result<(), Error> {
    let file = File::create(output)?;
    let mut zip = ZipWriter::new(&file);

    // Add screenshots
    for (i, action) in recording.actions.iter().enumerate() {
        if let Some(screenshot) = &action.screenshot_bytes {
            let entry_name = format!("screenshots/s{}.png", i);
            zip.add_entry(entry_name, screenshot)?;
        }
    }

    // Build trace.json
    let commands = recording.actions.iter().enumerate().map(|(i, a)| {
        TraceCommand {
            r#type: a.command,
            action: a.method,
            timestamp: a.timestamp,
            duration: a.duration_ms,
            screenshot_index: a.screenshot_bytes.is_some() ? Some(i as u32) : None,
            url: a.url,
            response: serde_json::from_str(&a.response_body)
                .unwrap_or(serde_json::Value::Null),
        }
    }).collect::<Vec<_>>();

    let trace = Trace {
        r#type: "trace",
        version: 2,
        pages: vec![TracePage {
            page_id: recording.session_id.unwrap_or("default"),
            title: "",
            url: "",
            commands,
        }],
    };

    let trace_json = serde_json::to_string_pretty(trace)?;
    zip.add_entry("trace.json", trace_json.as_bytes())?;

    zip.finish()?;
    Ok(())
}
```

- [ ] **Step 3: Write test that packages a mock recording**

```rust
fn test_package_trace_zip() {
    let recording = Recording {
        path: Path::new("/tmp/test-trace.zip"),
        actions: vec![RecordedAction {
            command: "click",
            url: "https://example.com",
            method: "POST",
            request_body: "{}",
            response_body: r#"{"value": {}}"#,
            status: 200,
            timestamp: 1000,
            duration_ms: 50,
            screenshot_bytes: Some(vec![0, 1, 2, 3]), // fake PNG
        }],
        session_id: Some("s1"),
        start_time: 1000,
    };

    let output = Path::new("test-output.zip");
    package(&recording, &output)?;
    assert(output.exists());
    // Cleanup
    std::fs::remove(&output)?;
}
```

---

### Task 6: Wire --pipe and --record into CLI

**Files:**
- Modify: `rust/src/main.rs`

- [ ] **Step 1: Add --pipe and --record flags to Cli struct**

```rust
#[clap(long)]
pipe: bool,

/// Path for trace recording ZIP. Implies --pipe.
#[clap(long, value_parser)]
record: Option<String>,
```

- [ ] **Step 2: Add pipe mode dispatch after existing setup**

```rust
// In main(), after selenium_manager.setup():
if cli.pipe || cli.record.is_some() {
    let driver_path = ...;  // from setup result
    let browser_path = ...;
    let pipe = PipeMode::new(PipeConfig {
        webdriver_url: format!("http://localhost:{}", webdriver_port),
        webdriver_ws_url: None,
        record_path: cli.record.map(|p| Path::new(&p).to_path_buf()),
    });
    let port = pipe.start()?;
    println!("{{\"proxy_url\":\"http://localhost:{}\",\"driver_path\":\"...\"}}", port);
    // Block until pipe exits (driver quit → proxy shutdown)
    pipe.wait();
    return;
}
```

- [ ] **Step 3: Test CLI with --help shows new flags**

```bash
cargo run -- --help 2>&1 | Select-String "pipe"
# Expected: shows --pipe and --record flags
```

---

### Task 7: Integration test — pipe + record with real chromedriver

**Files:**
- Create: `rust/tests/pipe_integration_tests.rs`

- [ ] **Step 1: Write integration test**

```rust
// This test requires chromedriver in PATH or managed by SM.
// It verifies the full flow: proxy → forward → screenshots → trace.zip.

fn test_pipe_record_with_chromedriver() {
    // 1. Start selenium-manager --pipe --record test.zip --browser chrome
    // 2. Get proxy URL from stdout
    // 3. Send POST /session to proxy (creates WebDriver session)
    // 4. Send POST /session/{id}/url {"url": "https://example.com"}
    // 5. Send GET /session/{id}/title
    // 6. Send DELETE /session/{id}
    // 7. Verify test.zip was created
    // 8. Verify trace.json and screenshots/ inside the zip
}
```

- [ ] **Step 2: Run integration test**

```bash
# Requires Chrome installed
cargo test --test pipe_integration_tests -v 2>&1 | Select-String "PASS"
```

---

### Task 8: Update Bazel BUILD for new files

**Files:**
- Modify: `rust/BUILD.bazel`

- [ ] **Step 1: Verify new .rs files are picked up**

The `glob(["src/**/*.rs"], exclude = ["main.rs"])` in `rust_library` already captures new files. No changes needed. Verify with:

```bash
bazelisk build //rust:selenium_manager 2>&1 | tail -3
```

---

### Non-goals (separate plans)

The following are explicitly **not** in this plan:
- Binding changes (Java, Python, JS, .Net, Ruby) — each gets its own implementation plan
- Video recording (Firefox screencast) — needs BiDi screencast, post-MVP
- Chunked recording — post-MVP
- HTML snapshots — post-MVP
- Direct stdin/stdout protocol (non-HTTP) — the design uses HTTP proxy