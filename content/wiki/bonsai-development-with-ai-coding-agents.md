---
title: "Bonsai development with AI coding agents"
url: "/bonsai-development-with-ai-coding-agents/"
parent: "/bonsai/"
categories: ["Bonsai", "AI coding agents"]
---

This page is a log of practical lessons from using AI coding agents — Claude Code, GitHub Copilot agent mode, Aider, Cursor and similar — to work on [Bonsai](/bonsai/). It is written after the fact, by the people who hit the problem.

It is deliberately not advice about prompting. The failures worth recording are rarely about the quality of generated code, which a review and a test suite will catch. They are about the **environment**, and a Bonsai development setup is an unusually easy one to damage: a Blender extension whose bundled Python wheels are managed by Blender itself, whose source is linked in from a Git checkout, and whose branches are long-lived and stacked. None of that is visible in the code an agent is looking at, and an agent asked to run a test will cheerfully reconstruct what it thinks your machine looks like.

Each entry says what was attempted, what actually happened, and the rule that would have prevented it. Please add your own.

## Practices

These are the accumulated rules from the entries below. They are written as instructions you can paste into an agent's project instructions or `AGENTS.md`/`CLAUDE.md` file.

1. **Read a known-good instance before changing an environment.** If a second, working installation exists — another Blender version, a colleague's setup, a container — inspect it first and match it. Reconstructing an environment from memory or documentation is how agents produce confidently wrong layouts.
2. **A sandbox that links back to real files is not a sandbox.** Symlinks, junctions, bind mounts and shared caches all mean a "throwaway" profile can delete production data. Copy, or point at nothing at all.
3. **Know what authority a tool has, not just the operation you want from it.** Package and extension managers reconcile state: they delete whatever is not in the manifest as readily as they install what is. Pointing one at an incomplete manifest is a delete instruction.
4. **Treat setup steps as changes.** Flags that feel read-only often are not. Anything that starts an application with a different configuration can cause that application to write configuration back.
5. **Permission and lock errors are a stop signal, not a retry signal.** They usually mean another process — often the user's live session — owns the files. Repeated attempts against a partially modified state make recovery harder than the original fault.
6. **Prefer read-only ways to answer a question.** "Will this cherry-pick cleanly?" is answerable with `git merge-tree`, which touches no working tree. "Does this branch contain that fix?" is a `git log -S` away. Reach for the destructive version only when the read-only one cannot answer it.
7. **Establish the restore path before the change, not after.** Rename rather than delete; snapshot the file first; note where the originals can be re-obtained. Agents are good at this when asked and rarely do it unprompted.
8. **Ask for verification of numbers and names.** File counts, versions, branch names and line numbers are exactly the details a language model will produce fluently and inaccurately. "Check that with a command before you tell me" is a cheap instruction.
9. **Have the agent work in a Git worktree.** Detached worktrees let an agent check out, merge and test other branches without touching the branch you have open, and without a stray commit landing on it.
10. **Say when a long-running application is open.** Much of the damage below traces to an agent operating on files a running Blender had loaded and locked.

## Incident log

### 2026-08-18 — Blender preferences replaced by factory defaults

**Attempted.** Running the Bonsai test suite headlessly, using `blender --factory-startup --python script.py` with `bpy.ops.preferences.addon_enable(...)` in the script.

**Happened.** Blender started from factory preferences, `addon_enable` marked preferences as modified, and because *Auto-Save Preferences* was enabled it wrote them out on exit — overwriting the real `userpref.blend` with factory defaults plus one add-on. Custom keymaps, theme, file paths and add-on preferences were lost.

**Rule.** `--factory-startup` affects what is loaded, not what is saved. Make the first line of any headless script `bpy.context.preferences.use_preferences_save = False`.

**Note on recovery.** Preferences copied from a newer Blender are a partial fix at best. A `.blend` file carries its own struct definitions and fields are matched by name, so most values transfer, but Blender's version patching only runs forward — old file into newer Blender. Fields that the newer version renamed or added have no counterpart in the older one and land as zero rather than as that version's default. Restoring 5.x preferences into 4.5 left theme entries such as tab outline and region background black. Export your theme as an XML preset (Preferences → Themes → `+`); XML is matched by property name and survives version changes far better than the binary preferences file.

### 2026-08-18 — Bonsai's bundled Python wheels deleted

**Attempted.** Testing a branch checked out in a separate worktree, without disturbing a running Blender. The agent built a scratch Blender profile via `BLENDER_USER_RESOURCES`, pointed its extension at the repository's `src/bonsai/bonsai` directory, and — so that the bundled dependencies would resolve — junctioned the scratch profile's `extensions/.local` at the real one.

**Happened.** Blender's extension wheel manager syncs the site-packages directory to the set of wheels declared by the enabled extensions. A development checkout of Bonsai has an **empty `wheels/` directory and no wheel list in its manifest**, so every installed package read as orphaned and was deleted — `ifcopenshell`, `shapely`, `lark`, `natsort` and roughly 180 others — through the junction, in the user's real installation. Only files locked by the running Blender survived.

**Rule.** Never present a source checkout to Blender as an installed extension, and never link a scratch profile's `.local` at a real one. See practices 2 and 3.

**Recovery.** The wheels are already on disk: an installed Bonsai keeps them in `extensions/user_default/bonsai/wheels/`. With Blender closed, reinstall them with Blender's own interpreter, which avoids the network and any version guessing:

```sh
for whl in <extensions>/user_default/bonsai/wheels/*.whl; do
  "<blender>/python/bin/python.exe" -m pip install --no-deps --no-index --upgrade \
    --target "<extensions>/.local/lib/python3.11/site-packages" "$whl"
done
```

If a package's `.pyd` is locked and it cannot be replaced, unzip that wheel and copy in only the files that are missing. Blender's own repair path — `bpy.ops.extensions.repo_refresh_all()` then disable and re-enable the add-on — does work, but only with no locks held; attempting it against a running Blender oscillated between installing and purging.

**Do not** `pip install` `bonsai-*.whl` or `ifcopenshell-*.whl` into that site-packages on a development machine. Those two entries are normally links to your Git checkout, and installing the packaged versions replaces them, after which Blender silently runs the release code while you edit the repository and wonder why nothing changes. A development setup looks like this, per Blender version:

| Path in `extensions/.local/lib/python3.x/site-packages/` | Points at |
| --- | --- |
| `bonsai` | `<repo>/src/bonsai/bonsai` |
| `ifcopenshell` | `<repo>/src/ifcopenshell-python/ifcopenshell` |

plus `extensions/user_default/bonsai/__init__.py` linked to the repository's copy. On Windows, `New-Item -ItemType SymbolicLink` needs administrator rights while `-ItemType Junction` does not and behaves the same for imports.

### 2026-08-18 — Fresh Git worktree missing compiled artifacts

**Attempted.** Running Bonsai's modal test suite against a branch checked out with `git worktree add`.

**Happened.** Blender opened a `FATAL ERROR: Unable to load Bonsai` dialog and the run hung until it timed out. `_ifcopenshell_wrapper.cp311-win_amd64.pyd` and `ifcopenshell_wrapper.py` are build artifacts that are not tracked by Git, so a fresh worktree does not have them, and `ifcopenshell` reports itself as not built for the platform.

**Rule.** A worktree contains tracked files only. Copy the untracked build artifacts across from your main checkout, or build them, before testing there. `ls` the main checkout against the worktree to find them.

## Adding an entry

Add a dated `###` heading under **Incident log**, newest last, and keep to the same four parts: what was attempted, what happened, the rule, and — where it exists — the recovery. Then generalise the rule into the **Practices** list above if it is not already covered.

The incidents here are Bonsai and Blender ones, but the practices are not specific to either. If entries start arriving from people working on other tools, move the **Practices** section to a general page and leave the Bonsai incidents on this one.

Two things make an entry worth reading. Name the mechanism rather than the symptom, so that somebody meeting a variation of it can recognise it. And record what recovered the situation, including the paths and commands, because the next person to hit it will be in a hurry.

## See also

- [Bonsai installation](/bonsai-installation/)
- [Free software libraries for AEC software development](/free-software-libraries-for-aec-software-development/)
