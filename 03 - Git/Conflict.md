---
tags:
  - Git
---
The Problem:
   You edited files on PC #1 and pushed to GitHub. Then you edited files on PC #2 WITHOUT pulling the changes from PC #1 first.
    Now both versions have different changes and Git doesn't know which to keep.

   Think of it like:
   •  PC #1 wrote "Version A" in a document and saved it online
   •  PC #2 still had the old document and wrote "Version B"
   •  Now Git sees two different versions and says "which one do you want?"

   How to Fix It:

   Option 1: Keep Remote Changes (Discard Local)

   bash
     git merge --abort          # Cancel the merge mess
     git reset --hard origin/main   # Force local to match remote exactly

   Option 2: Keep Local Changes (Discard Remote)

   bash
     git merge --abort          # Cancel the merge mess
     git push --force           # Force remote to match local (DANGEROUS!)

   Option 3: Merge Both (Manual Review)

   bash
     git pull                   # Download remote changes
     # Git shows conflicts
     # Edit files manually to keep what you want
     git add .
     git commit -m "Merged changes"

   Prevention:

   Always pull before editing:

   bash
     git pull    # Get latest changes FIRST
     # Then edit files
     git add .
     git commit -m "message"
     git push

   Golden Rule: PULL → EDIT → COMMIT → PUSH