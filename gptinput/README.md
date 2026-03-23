# GPT Input

This directory is a drop-zone for raw input from chats outside VS Code (other LLM sessions, exported notes, etc.).

Workflow:

- Put new, unprocessed files directly in `gptinput/`.
- After the agent processes them and promotes the useful content into canonical docs under `development/` and/or `product/`, move the original inputs into `gptinput/processed/`.

Notes:

- Content in `gptinput/` is not authoritative until it has been summarized and promoted.
- `gptinput/processed/` is an archive for traceability (instead of deleting inputs).