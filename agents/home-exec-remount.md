# Tor Browser home exec-remount: approaches A vs B

## Problem

security-misc mounts `/home` `noexec` (remount-secure, level 2-3). Tor Browser
must execute binaries under its home (`~/.tb`, `~/.cache/tb`), so something must
re-add `exec` to just those directories without weakening the rest of `/home`.

Two mechanisms exist in the tree. They differ in the ONE property that decides
whether the browser ever sees the exec bit: the mount namespace the remount lands
in.

## A -- tb-updater, PAM session (`usr/libexec/tb-updater/tb-home-remount-exec`)

- Runs from a PAM session hook (`usr/share/pam-configs/tb-home-exec`, `pam_exec`,
  `Session-Interactive-Only`, optional, priority 95) as the login session opens.
- The bind-remount therefore happens INSIDE the user's login-session mount
  namespace -- the same namespace the browser runs in -- so it is always visible
  to the browser.

## B -- tb-starter, launch time (`usr/libexec/tb-starter/remount-exec`)

- `usr/bin/torbrowser` calls `check_noexec_do` at launch and, if the home is
  `noexec`, runs `leaprun tb-starter-remount-exec` -> `remount-exec` as root via
  privleap.
- That root process lives in privleapd's init mount namespace, NOT the caller's
  login-session namespace, so its bind-remount lands in the init namespace.

## Decisive fact: pam_namespace

security-misc's pam_namespace gives each interactive login its OWN unshared mount
namespace (it polyinstantiates `/tmp`, `/var/tmp`, `/dev/shm`). A mount made in
the init namespace AFTER that unshare is not visible inside the session namespace.

Empirically confirmed on a pam_namespace host (sandbox e2e): B's exact two `mount`
commands, run as root in the init namespace, do NOT appear inside the login
session's namespace. So on any pam_namespace system B cannot deliver exec to the
browser; A can, because it runs in-session by construction.

Blast radius is NOT the differentiator: B is already hardcoded to
`/home/user/.tb`, so both approaches are narrow (one account, the `.tb` tree). No
amount of blast-radius scoping makes an init-namespace mount appear in a session
namespace -- only running in-session does.

## Comparison

| Dimension | A -- tb-updater, PAM session | B -- tb-starter, launch time |
|---|---|---|
| Where the remount runs | User's login-session mount ns (`pam_exec` at session open) | root in privleapd's init mount ns (`leaprun`) |
| Works with pam_namespace | Yes -- mount is in the browser's own session ns, by construction | No (e2e-confirmed) -- init-ns mount is invisible in the unshared session ns |
| Trigger / timing | At login, once | At browser launch, on demand (`check_noexec_do`) |
| Privilege | PAM session (root while the session opens) | root via privleap + sudoers |
| Account scope | `tb_home_exec_user` (default `user`); exits if `PAM_USER` differs | Hardcoded `/home/user/.tb` -- already `user`-only |
| Dirs made exec | `~/.tb` and `~/.cache/tb` | `~/.tb` only (+ `/var/cache/tb-binary/.tb` on Qubes TemplateVM) |
| Qubes | Exits early (`/run/qubes`) -- not used | Supported (TemplateVM path) |
| Safety guards | Symlink refusal, chown, `mkdir -p`, noexec-gated | noexec-gated (`mount \| grep noexec`) |

## Direction

Unify on A. B's launch-time init-namespace remount cannot reach a pam_namespace
session regardless of tuning; A is pam_namespace-correct and already covers the
non-Qubes case. The remaining work is to re-wire A (or an A-shaped equivalent) for
the one case B uniquely handles (the Qubes TemplateVM binary-cache dir), then
retire B's remount path.
