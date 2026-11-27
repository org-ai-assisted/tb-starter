# Tor Browser home exec-remount

## Problem

security-misc mounts `/home` `noexec` (remount-secure, level 2-3). Tor Browser
must execute binaries under its home (`~/.tb`, `~/.cache/tb`), so something must
re-add `exec` to just those directories without weakening the rest of `/home`.

The property that decides whether the browser ever sees the exec bit is the mount
namespace the remount lands in.

## Decisive fact: pam_namespace

security-misc's pam_namespace gives each interactive login its OWN unshared mount
namespace (it polyinstantiates `/tmp`, `/var/tmp`, `/dev/shm`). A mount made in
the init namespace AFTER that unshare is not visible inside the session namespace.

Empirically confirmed on a pam_namespace host (sandbox e2e): a bind-remount run as
root in the init namespace does NOT appear inside a login session's namespace. So
the exec-remount MUST run in-session, or the browser (which runs in the session
namespace) never sees it.

## Current design -- one mechanism, in-session

- Shared core: `tb-updater`'s `usr/libexec/tb-updater/tb-remount-exec.bsh`
  (symlink guard, ownership prep, per-directory noexec->exec remount). Sourced by
  both triggers below; no second copy.
- Non-Qubes and Qubes persistent: `tb-updater`'s PAM session hook
  `usr/libexec/tb-updater/tb-home-remount-exec` (`usr/share/pam-configs/tb-home-exec`,
  `pam_exec`, priority 95) runs as the login session opens, INSIDE the session's
  mount namespace, so the remount is visible to the browser by construction.
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
pam_namespace host that mount is invisible to the browser's session namespace, so
it could not deliver exec. Blast radius was not the issue -- it was already
hardcoded to `/home/user/.tb`; the namespace was. It has been removed, along with
its privleap action and sudoers grant; `tb-starter` now `Depends` on `tb-updater`.

## Comparison

| Dimension | A -- tb-updater, PAM session (chosen) | B -- tb-starter, launch time (retired) |
|---|---|---|
| Where the remount runs | User's login-session mount ns (`pam_exec` at session open) | root in privleapd's init mount ns (`leaprun`) |
| Visible to the browser under pam_namespace | Yes -- same session ns, by construction | No (e2e-confirmed) -- init-ns mount invisible in the session ns |
| Trigger / timing | At login, once (plus Qubes DispVM at boot) | At browser launch, on demand |
| Privilege | PAM session (root while the session opens) | root via privleap + sudoers |
| Account scope | `tb_home_exec_user` (default `user`) | Hardcoded `/home/user/.tb` |
| Dirs made exec | `~/.tb` and `~/.cache/tb` | `~/.tb` only |

## Caveat still to verify

A's PAM hook now also covers Qubes (the `/run/qubes` early-exit was removed).
Whether `pam_exec` actually fires on a Qubes AppVM login, and runs after the
DispVM boot-time binds, must be confirmed by a real Qubes AppVM boot test before
this is relied on there.
