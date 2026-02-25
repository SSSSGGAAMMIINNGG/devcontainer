# devcontainer

A devcontainer for Elixir and Phoenix development with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and optionally [Tidewave](https://tidewave.dev/).

The container runs Claude with `--dangerously-skip-permissions` behind a network firewall that restricts all outbound traffic to an allowlist of domains. This lets Claude operate autonomously without permission prompts while preventing it from reaching arbitrary endpoints.

## Quick start

> Requires the [devcontainer CLI](https://github.com/devcontainers/cli) (`npm install -g @devcontainers/cli`).

1. Copy the `.devcontainer/` directory into your Elixir project root.
1. Create a root `Makefile` that includes the devcontainer Makefile: `echo 'include .devcontainer/Makefile' > Makefile`
1. Add project-specific environment variables in `devcontainer.json`
1. Customize the allowed domains in `allowed-domains.txt`
1. Run `make dc.up` to start the container, then `make dc.shell` to open a shell inside it.
1. Run `make dc.claude` to start Claude in unsafe mode. In another terminal, start your app with `mix phx.server` and in yet another terminal, run `make dc.tidewave` to expose tidewave at [localhost:9833](http://localhost:9833)

## Makefile commands

Run `make list` to see all available commands.

## Adding allowed domains

All outbound HTTP/HTTPS traffic from the container is transparently intercepted by [Squid](https://www.squid-cache.org/) and filtered against a domain allowlist. Unlike a traditional forward proxy setup, this uses iptables `REDIRECT` rules to catch **all** traffic regardless of whether the process respects `HTTP_PROXY` environment variables.

Edit `.devcontainer/allowed-domains.txt` to add or remove domains:

```txt
# A leading dot matches the domain and all subdomains
.example.com          # matches example.com, api.example.com, etc.

# Without a leading dot, only the exact domain is matched
cdn.example.com       # matches cdn.example.com only
```

After changing the file, rebuild the container with `make dc.rebuild`.

## Adding environment variables

Environment variables are configured in `.devcontainer/devcontainer.json` in two sections:

### `remoteEnv` 

For secrets and values that should refresh from your host on each container restart:

```jsonc
"remoteEnv": {
  "MY_API_KEY": "${localEnv:MY_API_KEY}",
  "DATABASE_HOST": "host.docker.internal",
},
```

The `${localEnv:VAR_NAME}` syntax pulls the value from your host machine's environment. Set these variables in your shell profile (e.g., `~/.zshrc`) or use `direnv` or `dotenv`.

### `containerEnv`
For configuration that is baked in at container creation time and does not change between restarts:

```jsonc
"containerEnv": {
  "MIX_ENV": "dev",
},
```

> **Note:** The proxy works transparently via iptables — no `HTTP_PROXY`/`HTTPS_PROXY` environment variables are needed.

## Disabling Tidewave

To disable Tidewave, make two changes:

1. In `.devcontainer/Dockerfile`, set the build arg to `false`:

   ```dockerfile
   ARG INSTALL_TIDEWAVE=false
   ```

2. In `.devcontainer/devcontainer.json`, remove the Tidewave port mapping from `runArgs`:

   ```jsonc
   // Before
   "runArgs": ["--cap-add=NET_ADMIN", "--cap-add=NET_RAW", "-p", "4000:4000", "-p", "9833:9832"],

   // After
   "runArgs": ["--cap-add=NET_ADMIN", "--cap-add=NET_RAW", "-p", "4000:4000"],
   ```

Then rebuild with `make dc.rebuild`.

## How the firewall works

The container uses a layered approach to make `--dangerously-skip-permissions` safer:

### Network isolation

On container start, `init-firewall.sh` configures Squid and iptables:

1. **Start Squid** in transparent intercept mode with two ports: 3129 for HTTP (reads `Host` header) and 3130 for HTTPS (reads SNI hostname from TLS ClientHello via peek-and-splice — no decryption, no CA cert).
2. **Redirect all outbound HTTP/HTTPS traffic** via iptables `REDIRECT` rules — port 80 → Squid (3129), port 443 → Squid (3130). Squid's `proxy` user is exempted to prevent redirect loops.
3. **Filter requests by domain** using the allowlist in `allowed-domains.txt` (loaded directly by Squid).
4. **Allow non-web traffic** that the container needs: DNS (port 53), localhost, and the Docker host network.
5. **Set a default DROP policy** on INPUT, OUTPUT, and FORWARD chains — anything not explicitly allowed is blocked.
6. **Verify the rules** by confirming that `example.com` is blocked and `api.github.com` and `claude.ai` are reachable.

If any verification check fails, the container will not start.

### Project-level deny rules

On container start, `link-claude-project.sh` creates a Claude project-level `settings.json` that denies reading `.env` files:

```json
{
  "permissions": {
    "deny": [
      "Read(path:**/.env)",
      "Read(path:**/.env.*)"
    ]
  }
}
```

This file is written into the symlinked project directory so the rules apply both inside the container and on your host.

### Known limitations

The firewall reduces the attack surface significantly but is not a complete sandbox:

- **No general sudo:** The `dev` user only has scoped sudo access for the firewall init script. Claude cannot escalate privileges to flush iptables rules or modify system configuration. If you need general sudo for ad-hoc tasks, you can add it back in the Dockerfile, but this weakens the firewall guarantee.
- **Runtime environment variables:** The `.env` deny rules prevent reading `.env` *files*, but secrets injected via `remoteEnv` are still visible through `printenv` or `/proc/self/environ`. Avoid putting highly sensitive secrets in `remoteEnv` if this is a concern.
- **Non-HTTP protocols:** The firewall only restricts ports 80 and 443. Traffic on other ports (other than DNS and localhost) is blocked by the default DROP policy, but if you add custom allow rules, those channels are unfiltered.
- **SNI-based filtering:** HTTPS filtering relies on the SNI extension in the TLS ClientHello. Connections without SNI (rare) will be blocked by default.

## Design decisions

**npm install over the native Claude installer:** Claude Code is installed via `npm install -g @anthropic-ai/claude-code` rather than the native install script. The npm install is faster, produces a cacheable Docker layer, and avoids the native installer's interactive prompts and heavier runtime.

**Squid with iptables REDIRECT over HTTP_PROXY env vars:** A traditional forward proxy setup relies on processes respecting `HTTP_PROXY`/`HTTPS_PROXY` environment variables. Any process that makes direct connections (e.g., Erlang's `:httpc`, Typst's package manager) would bypass the proxy entirely. Instead, iptables `REDIRECT` rules transparently send all outbound port 80/443 traffic to Squid's intercept ports. Squid filters HTTP by `Host` header and HTTPS by SNI hostname using peek-and-splice (no TLS decryption, no CA certificate needed).

**Symlinked Claude project directory:** The container mounts `~/.claude` from the host and symlinks the container's workspace project path to the host's project path. This means Claude's conversations, memory, and settings persist across container rebuilds and are shared between host and container sessions.

**Mapped Tidewave port (9833 instead of 9832):** The container maps port 9832 (Tidewave's default inside the container) to port 9833 on the host to avoid clashes with a local Tidewave installation.

## Customizing the base image

The Dockerfile uses `hexpm/elixir` as the base image. To change the Elixir or OTP version, edit the build args at the top of `.devcontainer/Dockerfile`:

```dockerfile
ARG ELIXIR_VERSION=1.19.5
ARG OTP_VERSION=28.3.2
ARG DEBIAN_VERSION=trixie-20260202-slim
```