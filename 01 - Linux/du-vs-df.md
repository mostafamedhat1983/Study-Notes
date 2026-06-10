---
tags:
  - Linux
---
# du vs df

- `du` (disk usage) shows how much space a file or directory is using. It is useful when checking which folder is consuming disk space
- `df` (disk free/filesystem) shows total, used, and available space for a mounted filesystem or partition. It is useful when checking whether a disk or partition is full.

## Quick difference

- Use `du` for folders and files.
- Use `df` for filesystems or partitions.

## Deleted files note

Sometimes `df` shows a filesystem as full while `du` does not show matching usage.[] A common reason is a deleted file that is still open by a running process; the file disappears from directory listings, so `du` cannot count it, but the filesystem space remains allocated, so `df` still includes it until the process closes the file or restarts

## Handy commands

```bash
df -h
du -sh /var/log
lsof | grep deleted
```

- `df -h` checks overall filesystem usage.[
- `du -sh /path` checks the size of a specific directory.
- `lsof | grep deleted` helps find deleted files that are still open by processes.[
