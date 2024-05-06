# ssh-agent-lib

[![CI](https://github.com/wiktor-k/ssh-agent-lib/actions/workflows/rust.yml/badge.svg)](https://github.com/wiktor-k/ssh-agent-lib/actions/workflows/rust.yml)
[![Crates.io](https://img.shields.io/crates/v/ssh-agent-lib)](https://crates.io/crates/ssh-agent-lib)

A collection of types for writing custom SSH agents as specified by the [SSH Agent Protocol Internet Draft](https://datatracker.ietf.org/doc/html/draft-miller-ssh-agent).

This makes it possible to utilize remote keys not supported by the default OpenSSH agent.

## Example

The following example starts listening on a socket and processing requests.
On Unix it uses `ssh-agent.sock` Unix domain socket while on Windows it uses a named pipe `\\.\pipe\agent`.

```rust,no_run
#[cfg(not(windows))]
use tokio::net::UnixListener as Listener;
#[cfg(windows)]
use ssh_agent_lib::agent::NamedPipeListener as Listener;
use ssh_agent_lib::error::AgentError;
use ssh_agent_lib::agent::{Session, listen};
use ssh_agent_lib::proto::{Identity, SignRequest};
use ssh_key::{Algorithm, Signature};

#[derive(Default, Clone)]
struct MyAgent;

#[ssh_agent_lib::async_trait]
impl Session for MyAgent {
    async fn request_identities(&mut self) -> Result<Vec<Identity>, AgentError> {
        Ok(vec![ /* public keys that this agent knows of */ ])
    }

    async fn sign(&mut self, request: SignRequest) -> Result<Signature, AgentError> {
        // get the signature by signing `request.data`
        let signature = vec![];
        Ok(Signature::new(
             Algorithm::new("algorithm").map_err(AgentError::other)?,
             signature,
        ).map_err(AgentError::other)?)
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    #[cfg(not(windows))]
    let socket = "ssh-agent.sock";
    #[cfg(windows)]
    let socket = r"\\.\pipe\agent";

    let _ = std::fs::remove_file(socket); // remove the socket if exists

    listen(Listener::bind(socket)?, MyAgent::default()).await?;
    Ok(())
}
```

Now, point your OpenSSH client to this socket using `SSH_AUTH_SOCK` environment variable and it will transparently use the agent:

```sh
SSH_AUTH_SOCK=ssh-agent.sock ssh user@example.com
```

On Windows the path of the pipe has to be used:

```sh
SSH_AUTH_SOCK=\\.\pipe\agent ssh user@example.com
```

For more elaborate example see the `examples` directory or [crates using `ssh-agent-lib`](https://crates.io/crates/ssh-agent-lib/reverse_dependencies).

## License

This project is licensed under either of:

  - [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0),
  - [MIT license](https://opensource.org/licenses/MIT).

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this crate by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.

### Note

This library has been forked from [sekey/ssh-agent.rs](https://github.com/sekey/ssh-agent.rs) as the upstream seems not be maintained (at least as of 2022).
The library was previously licensed under MIT, however in [#36], we relicensed it to MIT/Apache 2.0.

Contributors gave their approval for relicensing [#36] [screenshot]

[#36]: https://github.com/wiktor-k/ssh-agent-lib/pull/36
[screenshot]: http://web.archive.org/web/20240408190456/https://github.com/wiktor-k/ssh-agent-lib/pull/36

What remains from the original library is considered minor and does not count for copyright assignment.
