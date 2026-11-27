# Tor Browser home exec-remount -- test plan

Companion to [home-exec-remount.md](home-exec-remount.md). Tracks what is verified
and what still needs a live pam_namespace host (operator-owned; cannot run from the
dev VM).

## Already verified (automated, committed)

- Shared helper `tb-remount-exec.bsh` unit tests (`dist-ai`
  `tb-updater-tests/test_remount_exec_lib.py`): symlink refusal; privilege-dropped
  create via `runuser` (no root mkdir/chown); `apply` remounts only when noexec,
  carries explicit `exec`, keeps `nosuid,nodev`, skips a redundant self-bind, no-ops
  when already exec or on findmnt failure.
- PAM hook structure (`tb-updater-tests/test_home_remount_hook_ns_guard.py`):
  namespace guard present and BEFORE the remount; slave-propagation hardening
  between the guard and the remount; creation only via the helper, no direct root
  mkdir/chown. Each FAILS on the pre-fix hook (canary-verified).
- `check_noexec` enforcement (`tb-starter-tests/check_noexec_do_test.sh`): blocks
  launch (exit 5) on an inherited noexec home; allows an exec home.
- Hook behavior with stubbed namespace values (dev-VM probe, not committed): init
  namespace -> 0 mount actions (skips); unshared -> `make-rslave` then create +
  remount. Confirms the guard gates correctly.

## Remaining -- requires a live pam_namespace host

pam_namespace is not merged/deployed yet. These cannot be stubbed; run once it ships.

### Ordering and activation

- [ ] Confirm `pam_exec` (pam-config `tb-home-exec`, priority 95) actually fires on
      an interactive login for the Tor Browser account.
- [ ] Confirm pam_namespace runs EARLIER than this hook in the session stack, so the
      mount namespace is already unshared when the hook runs. If it runs later, the
      guard sees the init namespace and SKIPS (fail-safe, but the browser gets no
      exec home) -- fix the stack ordering.

### Functional (pam_namespace active, /home noexec at remount-secure level 3)

- [ ] `~/.tb` and `~/.cache/tb` are exec inside the login session; `nosuid,nodev`
      kept; the rest of `/home` stays noexec.
- [ ] Tor Browser launches and execs its binaries from `~/.tb` (no exit-5 block).
- [ ] First-run path: with `~/.tb` absent, the hook pre-creates it (as the account)
      and the first `update-torbrowser` install lands in an exec mount.

### Propagation / isolation (the make-rslave fix -- proven only in a stub)

- [ ] The exec exception is NOT visible in the init namespace: check `findmnt` in
      PID 1's mount namespace does NOT show `~/.tb` exec (no propagation back).
- [ ] The exec exception is NOT visible in a SECOND concurrent session of another
      account.
- [ ] No mount-table growth in init across repeated logins (no leaked binds).

### Fail-safe (pam_namespace inactive)

- [ ] On a host without pam_namespace, the hook is a deliberate no-op (guard skips)
      and no system-wide remount appears; Tor Browser launch is blocked by
      `check_noexec` (exit 5) with the clear error.

### Robustness

- [ ] Pre-existing root-owned `~/.cache`: the hook skips that dir (best-effort
      create) and does NOT abort; it still applies the exception to `~/.tb`.
- [ ] Planted symlink at `~/.tb` / `~/.cache` / `~/.cache/tb`: refused; no
      bind-mount or create through it.

### Qubes DispVM (unchanged helper path, confirm no regression)

- [ ] `tb-updater-dispvm.service` still binds the read-only `/var/cache/tb-binary`
      into the home and the DispVM's Tor Browser execs from it (the shared `apply`
      is unchanged; `prepare_owner` de-privilege affects only the PAM hook, not
      dispvm's own inline create).

## Packaging (tb-starter upgrade path)

- [ ] Upgrade from a pre-change tb-starter removes the orphaned
      `/etc/sudoers.d/tb-starter` (via `rm_conffile`); no dpkg conffile prompt.
- [ ] `Depends: tb-updater` is intentionally unpinned; pin a lower bound at release
      time to the tb-updater version shipping the PAM hook (see PR #5 note).
