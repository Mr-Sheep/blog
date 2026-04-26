+++
title = "How a CS class at UIUC knows you are cheating"
date = "2026-04-13T21:56:18-05:00"

description = "How a CS class at UIUC uses your VSCode editing history and ai agent's server cache to file FAIR allegation against you."

tags = ["uiuc", "AI coding"]
+++

How a CS class at UIUC uses your VSCode editing history and ai agent's server cache to file FAIR allegation against you. 

I do not support the use of AI to do all the coursework, neither do I support scanning all student files to gather proof that you have used AI. This post is not intended to attacking anyone (I really liked and enjoyed most of the classes, A LOT). 

I am not a course staff for this class and none of them shared any information with me. Anything written here should be treated as pure speculations. 

## VSCode Timeline

Back in 2020, VSCode introduced a new feature called "Timeline", see release note [here](https://code.visualstudio.com/updates/v1_44#_timeline-view). 

This feature creates a json file under `User/History/xxxxxx` with an `entries.json` that contains the name of the source file being edited and the names of temporary snapshots of the file alongside with their timestamps:

```json
{
    "version": 1,
    "resource": "file:///xxx",
    "entries": [
        {
            "id": "TaJo.c",
            "timestamp": 1776134288492
        },
        {
            "id": "FiDo.c",
            "timestamp": 1776134299434
        }
    ]
}
```

for VSCode server, the directory is located at `~/.vscode-server/User/History`

## Various Agentic Coding tools

Similar to vscode timeline, claude/gemini/codex stores their logs on the server in the following, monitored using `auditd`:

```sh
~/.aider.chat.history.md
~/.claude
~/.codeium/windsurf
~/.codeium
~/.codex
~/.gemini
~/.antigravity-server
~/.vscode-server/data/User/History
root/.aider.chat.history.md
root/.claude
root/.codeium/windsurf
root/.codeium
root/.codex
root/.gemini
root/.antigravity-server
root/.vscode-server/data/User/History
```

## Fortune Teller

According to a screenshot, the internal tool used is called Fortune Teller, and from those screenshots, other courses might also be using them.

It does a few things: 

1. check you CPS

	This can potentially be done by running `diff` on different versions of your code found in `entries.json` and then calculate the file change against time spent.

2. check non UTF-8 characters