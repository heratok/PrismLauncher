# Feature: Offline-only launch mode

## Objective

Allow a fully offline (non-premium) workflow: add an offline account without a Microsoft
account, launch the game in normal mode instead of demo mode, and produce a Windows
artifact from a fork build without Azure code signing.

## Problem

PrismLauncher gates offline usage behind `AccountList::anyAccountIsValid()`, which filters
on `MinecraftAccount::ownsMinecraft()`. That predicate explicitly excludes
`AccountType::Offline`, so an offline account can never satisfy it. Five call sites depend
on it:

| Site | Effect |
| --- | --- |
| `launcher/ui/pages/global/AccountListPage.cpp:143` | Blocks adding an offline account. |
| `launcher/LaunchController.cpp:96` | Shows a "No Accounts" dialog and aborts the launch. |
| `launcher/LaunchController.cpp:114` | Skips the account picker when only offline accounts exist. |
| `launcher/LaunchController.cpp:141-157` | Forces `LaunchMode::Demo` for offline accounts. |
| `launcher/Application.cpp:1268` | Forces the first-run wizard with a login step. |
| `launcher/ui/pages/instance/VersionPage.cpp:402` | Blocks version download / instance update. |

## Scope

In scope:
- Add `AccountList::anyAccountIsUsable()`, true for an MSA account that owns Minecraft or
  for any offline account.
- Route the five call sites above through it.
- Make `LaunchController::decideLaunchMode()` return `LaunchMode::Normal` for offline
  accounts instead of falling back to demo.
- Add `push` and keep `workflow_dispatch` in `.github/workflows/build.yml`, restricted to
  Windows on manual/push events.

Out of scope:
- Changing `anyAccountIsValid()` semantics (other subsystems rely on entitlement checks).
- Offline UUID generation: already correct via `uuidFromUsername()` (offline v3 UUID).
- `ChooseOfflineNameDialog`: performs no account validation, no change required.
- Online servers, skin/cape fetching, or Microsoft authentication.

## Tasks

- [ ] T1 — Add `AccountList::anyAccountIsUsable()` (declaration + implementation).
- [ ] T2 — `AccountListPage.cpp`: drop the `anyAccountIsValid()` gate in
      `on_actionAddOffline_triggered()`.
- [ ] T3 — `LaunchController.cpp`: use `anyAccountIsUsable()` in `decideAccount()` and
      return `LaunchMode::Normal` for offline accounts in `decideLaunchMode()`.
- [ ] T4 — `Application.cpp`: do not force the first-run wizard when the only accounts are
      offline.
- [ ] T5 — `VersionPage.cpp`: allow version download with offline accounts.
- [ ] T6 — `.github/workflows/windows-offline.yml`: dedicated Windows fork workflow with
      `workflow_dispatch` and `push` to `feature/offline-mode`/`develop`/`master`/`main`,
      Release by default, reusing the existing setup/package actions.

Note on CI: `build.yml` is intentionally left untouched. It already exposes
`workflow_dispatch`, and its Azure Trusted Signing steps are already guarded by
`vars.CI_HAS_ACCESS_TO_AZURE` and a non-empty client ID, so a fork skips signing on its own.
Adding `push` there would instead run the full Linux/macOS/arm64 matrix on every push to the
fork, which is wasteful and can go red on missing signing secrets.

## Acceptance criteria

- With zero Microsoft accounts, "Add Offline" opens the name dialog and creates an account.
- Launching an instance with an offline account reaches `LaunchMode::Normal`; the "No
  Accounts" dialog and demo fallback are not reached.
- `--username` and offline v3 `--uuid` are passed unchanged by the existing pipeline.
- Pushing `feature/offline-mode` produces a Windows artifact with signing skipped.

## Checks

- No local Qt 6 / MSVC toolchain assumed; compilation is verified by CI
  (`build.yml`, Windows MinGW-w64 job).
- Static review of each call site to confirm `anyAccountIsValid()` is untouched and still
  used only where entitlement semantics are intended.

## Delivery

Single branch `feature/offline-mode` on fork `heratok/PrismLauncher`, work-unit commits,
no pull request to upstream (the gate is deliberate upstream policy).

## Progress

- [x] Repo forked and branch created from `develop` (390a240).
- [x] T1 — `anyAccountIsUsable()` added to `AccountList`.
- [x] T2 — `AccountListPage` gate removed.
- [x] T3 — `LaunchController` routed through `anyAccountIsUsable()` and offline launches in
      `LaunchMode::Normal`.
- [x] T4 — `Application::createSetupWizard()` no longer forces login when only offline
      accounts exist.
- [x] T5 — `VersionPage::on_actionDownload_All_triggered()` accepts offline accounts.
- [x] T6 — `windows-offline.yml` added.
- [x] Static review: no remaining `anyAccountIsValid()` call sites for play/launch gating;
      the method itself is untouched.
- [ ] CI verification pending: no local Qt 6 / CMake / MSVC toolchain on the dev machine, so
      compilation is proven by the `Windows (offline fork)` workflow run.
