---
icon: material/server-network
sidebar_icon: false
status: experimental
description: Running long-lived server processes before tests with filtersets and platform-specific scoping.
---

# Server wrappers

!!! experimental "Experimental: This feature is not yet stable"

    - **Enable with:** Add `experimental = ["server-wrappers"]` to `.config/nextest.toml`.

Nextest supports running long-lived _server wrappers_ before tests are run.
Unlike setup scripts, server wrappers keep running while matching tests execute,
and are stopped after test execution is complete.

Server wrappers can be scoped to:

* Sets of tests, using [filtersets](../filtersets/index.md).
* Specific platforms, using [`cfg` expressions](specifying-platforms.md).

Server wrappers are configured in two parts: _defining scripts_, and _setting
up rules_ for when they should be executed.

## Defining scripts

Server wrappers are defined using the top-level `scripts.server-wrapper`
configuration.

```toml title="Server wrapper definition in <code>.config/nextest.toml</code>"
[scripts.server-wrapper.my-api-server]
command = "cargo run -p test-server"
probe = { url = "http://127.0.0.1:8080/health" }
```

Commands can either be specified using Unix shell rules, or as a list of
arguments.

```toml
[scripts.server-wrapper.script1]
command = 'script.sh -c "Hello, world!"'
probe = { url = "http://127.0.0.1:8080/health" }

[scripts.server-wrapper.script2]
command = ["script.sh", "-c", "Hello, world!"]
probe = { url = "http://127.0.0.1:8080/health" }
```

### Specifying `relative-to`

Commands can be interpreted as relative to a particular directory by specifying
the `relative-to` parameter:

<div class="compact" markdown>

`"none"`
: Do not alter the command. This is the default value.

`"target"`
: The target directory.

`"workspace-root"`
: The workspace root.

</div>

Setting `relative-to` does not change the working directory of the server
wrapper, which is always the workspace root. It just prepends the directory to
the command if it is relative.

### Specifying `env`

A map of environment variables may be passed to a command by specifying the
`env` parameter.

```toml
[scripts.server-wrapper.my-api-server]
command = {
    command-line = "cargo run -p test-server",
    env = {
        PORT = "8080",
    },
}
probe = { url = "http://127.0.0.1:8080/health" }
```

Note that keys cannot begin with `NEXTEST`, as that is reserved for internal
use.

### Probe configuration

Server wrappers require a probe. The probe determines when the server is ready
for tests to start.

```toml
[scripts.server-wrapper.my-api-server]
command = "cargo run -p test-server"
probe = {
    url = "http://127.0.0.1:8080/health",
    interval = "200ms",
    timeout = "30s",
}
```

`url`
: The HTTP URL to poll. The server is considered ready when this URL returns a
  2xx status.

`interval`
: How often to poll. Defaults to `500ms`.

`timeout`
: Maximum time to wait before failing the run. Defaults to `60s`.

### Server wrapper configuration

Server wrappers can have the following additional options:

`slow-timeout`
: Marks startup as slow using the same structure as slow timeouts for tests.

`leak-timeout`
: Leak timeout configuration, using the same structure as tests and setup
  scripts.

`capture-stdout`
: `true` if stdout should be captured. Defaults to `true`.

`capture-stderr`
: `true` if stderr should be captured. Defaults to `true`.

## Setting up rules

In configuration, you can create rules for when to use server wrappers on a
per-profile basis. This is done via the `profile.<profile-name>.scripts` array
using the `server-wrapper` instruction.

```toml title="Basic rules"
[scripts.server-wrapper.api]
command = "cargo run -p api-server"
probe = { url = "http://127.0.0.1:8080/health" }

[[profile.default.scripts]]
filter = "package(my-integration-tests)"
server-wrapper = "api"
```

Server wrappers can also filter based on platform:

```toml title="Platform-specific rules"
[[profile.default.scripts]]
platform = { host = "cfg(unix)" }
filter = "test(/^integration::/)"
server-wrapper = "api"
```

## Execution behavior

A server wrapper _S_ is only executed if the current profile has at least one
rule where `filter` and `platform` match and `server-wrapper` is set to _S_.

Server wrappers are started before tests and become active only after their
probe succeeds. Matching tests then execute while the server wrapper remains
running.

### Output capture

Server wrapper stdout and stderr are captured while tests run. For each test
that uses a server wrapper, nextest records only the output produced during that
test's execution window and includes that output with the test's captured
output.
