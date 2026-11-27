# Tor Browser home exec-remount

## Problem

security-misc mounts `/home` `noexec` (remount-secure, level 2-3). Tor Browser
must execute binaries under its home (`~/.tb`, `~/.cache/tb`), so something must
re-add `exec` to just those directories without weakening the rest of `/home`.

The property that decides whether the browser ever sees the exec bit is the mount
namespace the remount lands in.

## Decisive fact: pam_namespace

security-misc's pam_namespace gives each interactive login its OWN unshared mount
namespace (it polyinstantiates `/tmp`, `/var/tmp`, `/dev/shm`). Whether a mount
made in the init namespace AFTER that unshare reaches the session namespace
depends on the propagation type of the target mount: with a private or slave
home mount it does NOT propagate in, so the browser never sees it.

Empirically confirmed on the tested host (sandbox e2e, default propagation): a
bind-remount run as root in the init namespace did NOT appear inside a login
session's namespace. A shared (`MS_SHARED`) home mount could propagate it in, so
B's visibility is config-dependent -- check the `/home` propagation type per
target before relying on it. That config-dependence is exactly why A is preferred:
running the remount IN-session makes it visible to the browser unconditionally,
regardless of propagation type.

## Current design -- one mechanism, in-session

- Shared core: `tb-updater`'s `usr/libexec/tb-updater/tb-remount-exec.bsh`
  (symlink guard, ownership prep, per-directory noexec->exec remount). Sourced by
  both triggers below; no second copy.
- Non-Qubes and Qubes persistent: `tb-updater`'s PAM session hook
  `usr/libexec/tb-updater/tb-home-remount-exec` (`usr/share/pam-configs/tb-home-exec`,
  `pam_exec`, priority 95) runs as the login session opens. It acts ONLY when the
  session's mount namespace is unshared from init (compares `/proc/self/ns/mnt` to
  `/proc/1/ns/mnt`) -- i.e. pam_namespace has already unshared it earlier in the
  stack. It then makes the session tree slave (`mount --make-rslave /`) before the
  remount, because an unshared namespace still inherits each mount's propagation
  type -- with `/home` shared (systemd default) the exec remount would otherwise
  propagate back to init and become system-wide. So locality is ENFORCED, not
  assumed. The remount is then session-local and visible to the browser. With
  pam_namespace inactive (shared init namespace) it is a NO-OP, because a remount
  there would be system-wide and permanent (`pam_exec` close_session does not
  umount); `tb-starter`'s launch-time `check_noexec` (exit 5) is the fail-safe
  until pam_namespace lands. The hook pre-creates the browser dirs PRIVILEGE-
  DROPPED (as the account, via `runuser`), never as root: a root `mkdir --parents`
  follows a user-planted parent symlink that the final-component `[ -L ]` guard
  cannot see and cannot win the race against -- a root create+chown LPE.
- Qubes DispVM: `tb-updater-dispvm.service` runs the DispVM data layout at boot
  (binding the read-only `/var/cache/tb-binary` cache into the home, `/rw/home`),
  using the same shared helper for the exec step. Lives in tb-updater because it
  provisions the tb-updater-managed binary cache, not because it launches.
- `tb-starter`'s `usr/bin/torbrowser` keeps only `check_noexec` -- the launch-time
  verification that FAILS the launch (exit 5) if the home is still `noexec`. It no
  longer performs a remount.

## Retired: launch-time init-namespace remount

The previous `tb-starter` path (`usr/libexec/tb-starter/remount-exec`, run as root
at browser launch via privleap) bind-remounted in privleapd's init namespace. On a
pam_namespace host that mount can be invisible to the browser's session namespace
(propagation-dependent), so it could not reliably deliver exec. Blast radius was
not the issue -- it was already
hardcoded to `/home/user/.tb`; the namespace was. It has been removed, along with
its privleap action and sudoers grant; `tb-starter` now `Depends` on `tb-updater`.

## Comparison

| Dimension | A -- tb-updater, PAM session (chosen) | B -- tb-starter, launch time (retired) |
|---|---|---|
| Where the remount runs | User's login-session mount ns (`pam_exec` at session open) | root in privleapd's init mount ns (`leaprun`) |
| Visible to the browser under pam_namespace | Yes -- same session ns, by construction | Not reliably -- e2e-invisible on the tested host; depends on the home mount's propagation type |
| Trigger / timing | At login, once (plus Qubes DispVM at boot) | At browser launch, on demand |
| Privilege | PAM session (root while the session opens) | root via privleap + sudoers |
| Account scope | `tb_home_exec_user` (default `user`) | Hardcoded `/home/user/.tb` |
| Dirs made exec | `~/.tb` and `~/.cache/tb` | `~/.tb` only |

## Caveats still to verify

- A's PAM hook now also covers Qubes (the `/run/qubes` early-exit was removed).
  Whether `pam_exec` actually fires on a Qubes AppVM login, and runs after the
  DispVM boot-time binds, must be confirmed by a real Qubes AppVM boot test before
  this is relied on there.
- The namespace guard only becomes FUNCTIONAL once pam_namespace is active AND
  ordered to run EARLIER than this hook in the session stack -- otherwise the
  namespace is not yet unshared when the hook runs and the exception is skipped
  (fail-safe, but the browser gets no exec home). pam_namespace is not merged yet;
  confirm the stack ordering when it ships. Until then the hook is a deliberate
  no-op and `check_noexec` governs launch.
