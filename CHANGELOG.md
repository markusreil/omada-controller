# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial omada-controller compose project (LAN/self-signed variant, upstream `mbentley/omada-controller` image).

### Changed

- `GEN_SELF_SIGNED_CERT` compose default flipped to `true` (LAN/self-signed deployment).
- Stopped publishing the controller's own HTTPS port (8043); browser UI is proxy-only so nginx terminates TLS.
- Proxy now talks to the controller backend over HTTPS (`VIRTUAL_PORT: 8043` + `VIRTUAL_PROTO: https`) because the controller 302-redirects its HTTP endpoint to its HTTPS port.
- Published the remaining device ports 29815–29817 (device info, remote terminal, device monitor).
- Default `OMADA_VERSION` changed from `latest` to `6.3` (maintainer deprecates `latest`, which still points at v5).

### Deprecated

### Removed

### Fixed

### Security
