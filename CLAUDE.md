# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Environment

This is a devcontainer-based development environment with:
- Ubuntu 24.04 base image
- Node.js 20
- Claude Code integration pre-configured

## Architecture

This repository serves as a basic devcontainer template for Claude Code development. The setup follows the guide referenced in README.txt (https://zenn.dev/taichi/articles/a4ea249f7d0f6b).

## Configuration

- `.devcontainer/devcontainer.json`: Main devcontainer configuration
- `.claude/settings.local.json`: Local Claude Code permissions (allows basic bash operations like find and ls)
- JSON files are configured for auto-formatting on save in VS Code