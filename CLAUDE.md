# CLAUDE.md — HAProxy Development Guide for AI Assistants

## Project Overview

HAProxy is a high-performance TCP/HTTP load balancer and proxy server written in C. Current version: **3.4-dev6** (active development). Licensed under GPL v2 (source) and LGPL v2.1 (headers).

## Build System

GNU Make-based. A `TARGET` is required.

```bash
# Typical Linux build
make TARGET=linux-glibc

# Full-featured build
make TARGET=linux-glibc USE_OPENSSL=1 USE_LUA=1 USE_PCRE2=1 USE_ZLIB=1

# Verbose build with warnings as errors
make V=1 ERR=1 TARGET=linux-glibc

# Clean
make clean
```

**Key variables:**
- `TARGET` — Required. Values: `linux-glibc`, `freebsd`, `openbsd`, `netbsd`, `solaris`, etc.
- `USE_*` — Feature flags: `USE_OPENSSL`, `USE_LUA`, `USE_PCRE2`, `USE_ZLIB`, `USE_QUIC`, etc. (40+ options)
- `CC` — Compiler (default: `cc`)
- `OPT_CFLAGS` — Optimization flags (default: `-O2`)
- `V=1` — Verbose output
- `ERR=1` — Treat warnings as errors
- `PREFIX` — Install prefix (default: `/usr/local`)

Modular make includes live in `include/make/` (verbose.mk, compiler.mk, errors.mk, options.mk).

## Repository Structure

```
src/             — C source files (~237 files, ~313K lines)
include/
  haproxy/       — Project headers (~336 files), organized by feature subdirs
  import/        — Third-party libraries (ebtree, ist.h, xxhash, mjson)
  make/          — Makefile includes
doc/             — Documentation (configuration.txt, coding-style.txt, etc.)
reg-tests/       — VTC regression tests (~37 categories)
tests/           — Unit tests (C and Python)
admin/           — Admin tools (halog, systemd, selinux, wireshark)
dev/             — Dev utilities (gdb scripts, coccinelle, tracing, flags)
scripts/         — Build/test/release scripts
addons/          — Add-ons (WURFL, etc.)
examples/        — Example configurations
.github/         — GitHub Actions CI workflows
```

### Source Code Organization

Source files in `src/` are organized by functional area:
- **Core:** `haproxy.c`, `proxy.c`, `server.c`, `backend.c`, `stream.c`, `connection.c`
- **Config parsing:** `cfgparse.c`, `cfgparse-global.c`, `cfgparse-listen.c`, `cfgparse-ssl.c`, `cfgparse-quic.c`, `cfgparse-tcp.c`
- **Protocols:** `proto_tcp.c`, `proto_http.c`, `proto_quic.c`, `h1.c`, `h2.c`, `h3.c`
- **SSL/TLS:** `ssl_sock.c`, `ssl_ckch.c`, `ssl_crtlist.c`, `ssl_ocsp.c`
- **CLI:** `cli.c`
- **Checks:** `check.c`, `tcpcheck.c`
- **Data structures:** `ceb*_tree.c` (CEBTree implementations)

Headers in `include/haproxy/` mirror source organization with feature subdirectories (balance/, cache/, checks/, compression/, connection/, filters/, http-*/, jwt/, log/, lua/, peers/, pki/, proxy/, quic/, sample_fetches/, server/, spoe/, ssl/, stats/, stick-table/, stream/, tcp-rules/, webstats/).

## Testing

### Regression Tests (VTC)

Tests use VTest (Varnish Test Case format) in `reg-tests/`.

```bash
# Run all regression tests (requires vtest binary)
make reg-tests VTEST_PROGRAM=/path/to/vtest

# Filter by type
make reg-tests REGTESTS_TYPES=default,bug,devel

# Run specific test
/path/to/vtest reg-tests/http-rules/some_test.vtc
```

Test categories (37 dirs): balance, cache, checks, compression, connection, http-capture, http-cookies, http-errorfiles, http-messaging, http-rules, http-set-timeout, jwt, log, lua, mailers, mcli, peers, pki, proxy, quic, sample_fetches, seamless-reload, server, spoe, ssl, startup, stats, stick-table, stickiness, stream, tcp-rules, webstats, and more.

### Unit Tests

```bash
make unit-tests
```

Located in `tests/unit/`. Includes C test programs and Python test scripts.

### CI/CD

GitHub Actions workflows in `.github/workflows/`:
- `vtest.yml` — Main regression test suite
- `musl.yml` — Alpine/musl builds
- `aws-lc.yml`, `quictls.yml`, `wolfssl.yml` — TLS library variants
- `openssl-master.yml` — OpenSSL bleeding edge
- `cross-zoo.yml` — Cross-compilation tests
- `windows.yml` — Windows/MSYS2 builds
- `contrib.yml` — Dev utilities build (flags, poll, hpack)
- `coverity.yml` — Static analysis
- `codespell.yml` — Spell checking
- `compliance.yml` — Compliance checks
- `quic-interop-*.yml` — QUIC interoperability tests

Cirrus CI (`.cirrus.yml`) handles FreeBSD builds.

## Coding Conventions

Full guide: `doc/coding-style.txt`. Key rules:

### Indentation & Formatting
- **Indentation:** Tabs only (1 tab = 8 spaces width)
- **Alignment:** Spaces only (for continued lines)
- **Never mix tabs and spaces** for indentation
- Opening brace on same line as function/control statement
- One statement per line
- Variable declarations at the start of functions

### Style Checking
```bash
# Use Linux kernel's checkpatch.pl
checkpatch.pl -q --max-line-length=160 --no-tree --no-signoff \
  --ignore=LEADING_SPACE,CODE_INDENT,DEEP_INDENTATION,ELSE_AFTER_BRACE
```

### Naming
- Lowercase with underscores for functions and variables
- Prefix functions with their module area (e.g., `ssl_sock_`, `cli_`, `proxy_`)
- Macros and constants in UPPER_CASE

### Comments
- Use `/* ... */` style (not `//`)
- File headers include copyright, author, and license

### Licensing
- `.c` files: GPL v2 or later
- `.h` files: LGPL v2.1 or later
- Include appropriate license header in new files

## Commit & Contribution Guidelines

Full guide: `CONTRIBUTING` (read carefully before submitting).

### Commit Messages
- Explain **what** changed, **why**, and **how**
- One logical change per commit
- Separate functional changes from style fixes
- Commit quality directly impacts backportability

### Patch Rules
- Test thoroughly before submitting
- Follow coding style strictly
- Include documentation updates when relevant
- Avoid unnecessary code reformatting in functional patches
- Split complex features into multiple, reviewable patches

## Key Documentation

| File | Content |
|------|---------|
| `doc/configuration.txt` | Full configuration reference (1.6 MB) |
| `doc/management.txt` | Administration & management guide |
| `doc/intro.txt` | Introduction and concepts |
| `doc/coding-style.txt` | Coding conventions |
| `doc/regression-testing.txt` | Testing guide |
| `doc/lua.txt` | Lua scripting reference |
| `doc/internals/` | Developer internals documentation |
| `CONTRIBUTING` | Contribution guidelines |
| `INSTALL` | Build and installation instructions |
| `BRANCHES` | Release cycle and branch management |

## Useful Scripts

| Script | Purpose |
|--------|---------|
| `scripts/run-regtests.sh` | Run regression tests |
| `scripts/run-unittests.sh` | Run unit tests |
| `scripts/build-ssl.sh` | Build OpenSSL variants |
| `scripts/build-vtest.sh` | Build vtest tool |
| `scripts/backport` | Backporting assistant |
| `scripts/create-release` | Release management |

## Common Development Tasks

### Adding a new configuration keyword
1. Add parsing in the appropriate `cfgparse-*.c` file
2. Add keyword registration in the relevant source module
3. Update `doc/configuration.txt`
4. Add regression test in `reg-tests/`

### Adding a new sample fetch or converter
1. Implement in the relevant `src/` module
2. Register via the keyword table
3. Document in `doc/configuration.txt`
4. Add tests

### Debugging
- GDB scripts in `dev/gdb/`
- Tracing utilities in `dev/trace/`
- Flag analysis tools in `dev/flags/`
- SSL key logging in `dev/sslkeylogger/`
