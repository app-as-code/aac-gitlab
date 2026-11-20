# Changelog

All notable changes to this deployment definition are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed

- README rewritten in English, with configuration reference and project icon.

## [0.0.1]

### Added

- Initial App-as-Code service definition for GitLab CE 18.9.1 with PostgreSQL 16 and Redis 7.4 dependencies.
- Web interface, Git over SSH, container registry and GitLab Pages endpoints.
- Optional OpenID Connect login (Authentik) alongside credential login.
- Sizing variables for PostgreSQL, Puma and Sidekiq.
