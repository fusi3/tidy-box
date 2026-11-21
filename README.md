# TidyBox

A Windows app that helps you empty an "inbox" folder. It looks at each loose file and suggests which of your folders it belongs in (or whether it can go), using a small AI model that runs **entirely on your PC**. Nothing is uploaded anywhere.

> **Status: early development.** Nothing works yet. The design lives in [PLAN.md](PLAN.md).

## The idea

- You keep a folder where new things land (Downloads, scans, WhatsApp images…).
- TidyBox reads your existing folder structure and learns where things usually go.
- It suggests a home for each item. You accept, change or skip. Nothing moves without your OK, deletes go to the Recycle Bin, and everything can be undone.
- A search box finds anything it moved.

It's a companion to [FolderBadges](https://github.com/fusi3/windows-folder-badges), which puts a count badge on your inbox folder so you know when it's time to tidy.

## Building

Needs the [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0).

```
dotnet build
dotnet test
dotnet run --project src/TidyBox.Cli
```

## Layout

| Folder | What's in it |
|---|---|
| `src/TidyBox.Core` | Indexing, reading files, suggestions. No UI. |
| `src/TidyBox.Cli` | Command-line tool for the dry run and the accuracy test |
| `tests/TidyBox.Tests` | Tests |
| `PLAN.md` | The design. The source of truth for what gets built. |

## Licence

MIT. See [LICENSE](LICENSE).
