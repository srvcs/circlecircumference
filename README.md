# srvcs-circlecircumference

The circle-circumference orchestrator of the srvcs.cloud distributed standard
library.

Its single concern: **geometry: circumference of a circle.** It owns the
*control flow* — composing a constant and a float primitive — but does no
arithmetic of its own. It asks [`srvcs-pi`](https://github.com/srvcs/pi) for the
value of pi, then uses [`srvcs-floatmultiply`](https://github.com/srvcs/floatmultiply)
twice: once to form the diameter `2 * radius`, and once to scale it by pi.

```
circlecircumference(radius):
    p = pi()                       # constant service, called with an EMPTY body
    d = floatmultiply(2, radius)   # the diameter
    return floatmultiply(p, d)     # pi * (2 * radius)
```

The result is an `f64` — a JSON number that may be fractional. For example
`circlecircumference(1) == 6.283185307179586` (i.e. `2 * pi`).

Validation is not handled here. This service never calls `srvcs-isnumber`
directly; instead its dependencies validate their own operands, and any `422`
they raise is forwarded verbatim.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` | Service identity, concern, and dependency list |
| `POST` | `/` | Compute the circumference of a circle of `radius` |
| `GET` | `/healthz` `/readyz` `/metrics` `/openapi.json` | srvcs service standard surface |

```sh
curl -s -X POST localhost:8080/ -H 'content-type: application/json' -d '{"radius": 1}'
# {"radius":1,"result":6.283185307179586}
```

Responses:

- `200 {"radius": r, "result": n}` — evaluated; `result` is a float.
- `422` — a dependency rejected the input (forwarded verbatim).
- `500` — a reachable dependency returned a `200` without a numeric `result`
  (a contract violation).
- `503` — a dependency is unavailable.

## Dependencies

- [`srvcs-pi`](https://github.com/srvcs/pi)
- [`srvcs-floatmultiply`](https://github.com/srvcs/floatmultiply)

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `SRVCS_BIND_ADDR` | `0.0.0.0:8080` | Bind address |
| `SRVCS_PI_URL` | `http://127.0.0.1:8090` | Base URL of `srvcs-pi` |
| `SRVCS_FLOATMULTIPLY_URL` | `http://127.0.0.1:8091` | Base URL of `srvcs-floatmultiply` |
| `SRVCS_ENV` | `development` | Environment label for logs |
| `RUST_LOG` | `info,tower_http=info` | Tracing filter |

## Local checks

```sh
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Orchestration tests stand up *computing* mock dependency services in-process —
they read the request body and return the real `pi` / `a * b`, so the
composition is genuinely exercised against the asserted cases (compared
approximately, since the result is a float). See
[`srvcs/platform`](https://github.com/srvcs/platform) for the shared standard.

> Note: the `cargoHash` in `flake.nix` is inherited from the template and must be
> refreshed with a `nix build` before the Nix gates pass.
