# Read-only Pre-flight Gate

Pre-flight gate for trading write operations. Run this **before** any other trade write pre-flight step once a write intent is confirmed — before `account order-chance`, `market ticker`, reading batch files, calculating TWAP slices, or looking up an order to cancel.

Rationale: `read_only` is a purely local value — the effective profile's `read_only` in `~/.bithumb/config.toml`, with no env var or live API involved. Checking it up front avoids wasting trading pre-flight work that would be thrown away when the write is ultimately blocked.

> **This gate is your responsibility — there is no code block behind it.** The runner does **not** intercept write-preflight reads; in read-only mode they still run and hit the network. What helps you is `config show --json`: when the effective profile is read-only it returns a `read_only_notice` (`blocked_for_writes`, `write_capable_profiles`, `guidance`). Read it up front and skip the trading pre-flight lookups yourself — running them wastes calls on a write that cannot execute. See CONTEXT.md *write-preflight read* and ADR 0014.

## Steps

1. **Confirm the request is a write operation.** Reads (`trade get`/`list`, `twap list`, balance/ticker queries) need no gate — skip straight to them. The trading write operations blocked at execution in read-only mode are:
   - `trade place`
   - `trade cancel`
   - `trade batch-place`
   - `trade batch-cancel`
   - `twap place`
   - `twap cancel`

2. **Once-per-session check.** If you have already checked the effective profile's `read_only` in this session and the profile has not changed, skip to the normal pre-flight steps — do not re-run the check.

3. **Read the effective profile's `read_only` (local, no network):**

   ```bash
   bithumb config show --json
   ```

   Read `effective_profile`, then `profiles[effective_profile].read_only`. An **unset** `read_only` means writes are allowed (proceed). `false` means proceed. `true` means the gate trips — go to step 4.

   (If the user named a profile for this order, check that one: `bithumb config show --profile <name> --json` and read `settings.read_only`. If `settings` is **null** the profile does not exist — do not proceed and do not treat the missing value as "writes allowed". Run `bithumb config show --json` to list the configured profiles, show them to the user, and ask for a valid name.)

4. **When `read_only` is true — stop, do not run trading pre-flight lookups.** The write would be blocked at execution anyway. Instead:
   - Tell the user the effective profile is read-only, so real orders are not possible with it.
   - **List the write-capable profiles** (those whose `read_only` is not true) from `bithumb config show`, so the user can pick one.
   - Ask the user to re-run with `--profile <name>`. **Do not auto-select a profile**. Name the candidates, let the user choose.

> **Never bypass read-only.** When the gate trips, do **not** work around it on the user's behalf. Specifically forbidden (see CONTEXT.md *Read-only bypass*): attaching `--profile <other>` yourself, changing `default_profile`, running `config set` to flip `read_only` to `false`, or re-issuing the same write under a different write-capable profile. The user enabled read-only on purpose; switching profiles or disabling read-only is the user's call, not yours. You may **list** write-capable profiles — selecting and running one is up to them.

Only when `read_only` is not true do you continue to the rest of the write pre-flight checklist.
