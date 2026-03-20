# Privacy Policy

**Autoresearch** is a Claude Code plugin that runs entirely on your local machine.

## Data Collection

This plugin does **not** collect, store, transmit, or share any user data. It has no server component, no analytics, no telemetry, and no network calls of its own.

## How It Works

The plugin provides instructions (markdown files) that guide Claude Code's behavior. All processing happens within your existing Claude Code session. Any API calls made during experiments (e.g., to LLM providers, search APIs) are initiated by your experiment scripts using your own API keys — the plugin itself makes no network requests.

## Third-Party Services

Experiments you write may call third-party APIs (OpenAI, Google Gemini, Tavily, etc.) using API keys you provide in your local `.env` file. These interactions are governed by those services' respective privacy policies, not this plugin.

## Contact

For questions, open an issue at https://github.com/tingyiy/autoresearch/issues.
