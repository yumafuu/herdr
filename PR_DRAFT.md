# feat: keybinds to cycle focus through agent panes

## Summary

Adds four new optional navigate-mode keybinds:

- `next_agent` / `previous_agent` — cycle keyboard focus through every pane herdr has detected as an agent pane.
- `next_blocked_agent` / `previous_blocked_agent` — same, but filtered to agent panes currently in the `AgentState::Blocked` state ("needs attention").

All four ship **unbound by default** so existing users see no behavior change. Users opt in by assigning a key in `~/.config/herdr/config.toml`.

## Motivation

When several agents run side by side across workspaces and tabs, jumping to "the next thing that's working" or "the next thing that needs me" is a multi-step navigation today (switch workspace → switch tab → focus pane). A single keystroke handles the common cases.

## Design decisions

- **Cycle scope: global, across workspaces and tabs.** When the target pane lives in another workspace/tab, the action switches active workspace, switches active tab, then focuses the pane. This matches the user's stated intent ("a single keybind to cycle focus through agent panes globally"). Workspace-scoped variants can be added later if there's demand.
- **Ordering: same as the sidebar's "all workspaces" agent panel.** Workspace order, then tab order within a workspace, then pane creation order within a tab (`tab.layout.pane_ids()`). The user sees the same list visually that they're cycling through, and the order is deterministic.
- **What counts as an "agent pane": a pane where `effective_agent_label()` is `Some(_)`.** Same definition the sidebar agent panel already uses (whether by detection or hook authority). Plain shells are skipped automatically.
- **What counts as a "blocked" agent pane: `pane.state == AgentState::Blocked`.** This is the same state that drives the "needs attention" toast/sound.
- **Wraparound:** yes, both directions, mirroring `next_tab` / `previous_tab`.
- **No detected agents:** silent no-op. Same for `next_blocked_agent` when no panes are blocked.
- **Currently focused pane is not in the target set:** `next_*` jumps to the first pane in the set; `previous_*` jumps to the last.
- **Binding scope: TerminalDirect** (matches `next_tab` / `focus_pane_*`). The combos work directly from terminal mode without the prefix, which is what users want for a quick "jump to the next agent" gesture.
- **Both navigate-mode and terminal-direct dispatch are wired**, so users can also bind them under their prefix if they prefer.

## Testing

- `cargo build` — clean.
- `cargo test --bin herdr` — 757 tests pass, including 7 new tests covering:
  - empty / no-agent no-op cases for both unfiltered and blocked variants,
  - cycling forward across workspaces and tabs (including a wrap),
  - cycling backward with wraparound,
  - jumping in from a non-agent focused pane,
  - blocked filter skipping non-blocked agent panes,
  - blocked filter wraparound.
- `cargo fmt --check` — clean.
- `cargo clippy --all-targets` — no new warnings introduced; pre-existing warnings only.
- Manual UI exercise was not performed in this branch; reviewer should sanity-check the cycle behavior interactively before merging.

## Out of scope / open questions for review

- Workspace-scoped variant (`next_agent_in_workspace`) was deliberately not added; a single global behavior is simpler and the existing per-workspace navigation (`next_tab`, etc.) already covers tab-level cycling.
- "Done unattended" panes (`Idle` + `seen=false`) get no dedicated keybind here. If that turns out to be a useful target, it can be added with the same filter pattern (`agent_pane_locations_filtered`).
- Default key suggestions in `CONFIGURATION.md` use `alt+,` / `alt+.` and `alt+<` / `alt+>` purely as illustrative examples; no defaults are shipped.
