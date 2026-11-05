# CLAUDE.md

## How to work (high-level mindset)

The marginal cost of completeness is near zero with AI. Do the whole thing. Do it right. Do it with tests. Do it with documentation. Do it so well that KD is genuinely impressed — not politely satisfied, actually impressed. Never offer to "table this for later" when the permanent solve is within reach. Never leave a dangling thread when tying it off takes five more minutes. Never present a workaround when the real fix exists. The standard isn't "good enough" — it's "holy shit, that's done."

Search before building. Test before shipping. Ship the complete thing. When KD asks for something, the answer is the finished product, not a plan to build it.

Time is not an excuse. Fatigue is not an excuse. Complexity is not an excuse. Boil the ocean. This is how we think about shipping.

You can outsource the typing. You cannot outsource the understanding. Before you call anything DONE you must be able to explain why the code is correct and exactly where it would break. Tests passing is not understanding. If you can't walk the failure modes out loud, you're not done, you're guessing.

## Branching - one branch per task

This section is non-negotiable and must never be removed. It runs first, before the triage block, because the triage block has to report the branch it produces.

Two facts hold at once: KD works with other people, so nothing lands on `main` directly; and several Claude Code sessions run on the same machine, in the same repo, at the same time.

- Branches are named `kd/<task-id>-<task-title>`.
- Create branches from `main` only, and always fetch the latest `main` before creating a new branch for a task.
- Multiple sessions may be started for the same task. They continue on the branch already created for that task rather than creating a new one.
- Because sessions share one machine, prefer a separate `git worktree` per session so concurrent sessions never fight over the same working tree, `.dart_tool/`, or `build/` output.

## Task sizing

Every task starts with a printed triage block, before any work. One exception, and only one: the setup block in "Branching" runs first, because the triage block reports the branch it creates. Four lines:

* Size: small | medium | large — why
* Tests: local (which ones) | full suite — why
* Platforms: which targets the change can affect (android | ios | web | macos | windows | linux | all) — why, and which ones will actually be verified
* Branch: `<branch name>` — see "Branching"

This block is mandatory and verbose on purpose. KD reads it to see what mode was picked and to tune these rules over time. A wrong mode is only correctable if the choice is visible. Never skip it, never bury it mid-report. The Branch line is there so that with several sessions running at once, KD can tell at a glance which one is about to touch what.

## Completion status protocol

At the end of every task, report one of:

* DONE — All steps completed. Evidence provided for every claim with tests + evals. Ready to merge.
* DONE_WITH_CONCERNS — Completed, but with issues KD should know about. List each concern with severity and a proposed follow-up.
* BLOCKED — Cannot proceed. State what's blocking and what was already tried.
* NEEDS_CONTEXT — Missing information required to continue. State exactly what's needed.

"Partially done" is not a status. Either the feature ships (DONE) or it doesn't (BLOCKED / NEEDS_CONTEXT). Honesty about incompleteness beats pretending.

For anything touching platform-specific code, a DONE report must list which platforms were actually run or built and which were only reasoned about. "Should work on iOS" is not evidence.

## Safety

* Never edit secrets.
* Never commit secrets. If `.env` is touched, verify `.gitignore` before any commit.
* Platform credential files are secrets too and must stay out of git: `android/key.properties`, `*.jks` / `*.keystore`, `android/app/google-services.json`, `ios/Runner/GoogleService-Info.plist`, `macos/Runner/GoogleService-Info.plist`, provisioning profiles and certificates, and any `--dart-define-from-file` env files containing real keys. Verify `.gitignore` covers them before any commit.
* Never hard-code API keys or tokens in Dart source. Inject them at build time with `--dart-define` / `--dart-define-from-file`, or load them from secure storage at runtime.
* Never skip pre-commit hooks with `--no-verify`. If a hook fails, fix the underlying issue.

## Development flow

- Define the task
- Create the branch
- Implement the change
- Commit the change to the branch
- Open a pull request

## Conventions

- Flutter and Dart versions: pinned in `pubspec.yaml` (`environment:` block) and, if present, `.fvmrc`. Use the pinned version; do not upgrade the SDK as a side effect of another task.
- Format every change with `dart format` before committing.
- `flutter analyze` must pass with zero issues. Lints come from `analysis_options.yaml` (`flutter_lints` or stricter); do not silence a lint with `// ignore:` without a comment explaining why.
- Prefer the Flutter/Dart SDK and first-party packages; justify every new dependency in the PR description (what it does, maintenance status, and which platforms it supports). A package that does not support all target platforms is not acceptable unless it is isolated behind an interface with a fallback.
- Follow [Keep a Changelog](https://keepachangelog.com/) conventions in `CHANGELOG.md`.
- Exported (public) identifiers need `///` Dart doc comments; avoid comments that restate what the code already says.
- Every line of code must have an English comment above it that clearly states what it does — be concise and clear. Applies to hand-written code only; generated files (`*.g.dart`, `*.freezed.dart`, `*.gr.dart`, `generated_plugin_registrant.*`, localization output) are exempt and are never edited by hand.
- Every function, method, and widget class must have an English comment above it that clearly states what it does — be concise and clear.
- Naming follows Effective Dart: `UpperCamelCase` for types, extensions, and enums; `lowerCamelCase` for members, variables, and parameters; `lowercase_with_underscores` for files, directories, and packages; no leading underscores except for library-private members.
- Keep widgets small and composable. Prefer `const` constructors wherever possible. Extract widgets into classes rather than helper methods that return `Widget`.
- Never put business logic in widgets. UI calls into state management; state management calls into repositories/services.
- Never call `BuildContext` across an async gap without checking `context.mounted`.
- User-facing strings go through localization (`l10n/*.arb`), never string literals in widgets.

### Cross-platform rules

This app targets Android, iOS, web, macOS, Windows, and Linux unless the project's `pubspec.yaml` / `README.md` narrows the list. Every change is written so that it works on all supported targets.

- Do not import `dart:io` in code reachable from web. Use `package:flutter/foundation.dart` (`kIsWeb`, `defaultTargetPlatform`) for branching, and conditional imports (`if (dart.library.io)` / `if (dart.library.js_interop)`) for platform-specific implementations behind a common interface.
- Do not use `dart:html`; use `package:web` and `dart:js_interop` for web-only code.
- Isolate platform differences behind an abstract interface in `lib/src/services/` with per-platform implementations. Widgets and business logic never check the platform directly, except for purely visual adaptation.
- Layouts must be responsive and adaptive: use `LayoutBuilder` / `MediaQuery.sizeOf` breakpoints, never fixed pixel sizes tied to a phone screen. Support touch, mouse, keyboard, and focus traversal. Use `.adaptive` widget constructors or `Theme.of(context).platform` where native feel matters.
- Plugins that need native permissions must declare them for every platform (Android `AndroidManifest.xml`, iOS/macOS `Info.plist` and entitlements, web `index.html` where applicable). A permission added on one platform and forgotten on another is a bug.
- Do not assume a file system, background execution, or native plugins exist on web. Gate them and provide a fallback.
- Persisted data and network calls go through the repository/service layer so storage and transport can differ per platform.
- Do not edit platform folders (`android/`, `ios/`, `macos/`, `windows/`, `linux/`, `web/`) unless the task requires it, and explain every platform-folder change in the PR description.
- Accessibility is part of "done": semantic labels, sufficient contrast, scalable text (test at large text scale), and screen-reader support.

## Commands

- Install dependencies: `flutter pub get`
- Run (pick a device): `flutter run -d <device-id>` (list devices with `flutter devices`)
- Run on web: `flutter run -d chrome`
- Run on desktop: `flutter run -d macos` | `flutter run -d windows` | `flutter run -d linux`
- Build: `flutter build apk` | `flutter build appbundle` | `flutter build ios --no-codesign` | `flutter build web` | `flutter build macos` | `flutter build windows` | `flutter build linux`
- Unit and widget tests: `flutter test`
- Single test file: `flutter test test/path/to/foo_test.dart`
- Integration tests: `flutter test integration_test -d <device-id>`
- Format: `dart format --set-exit-if-changed .` (fix with `dart format .`)
- Analyze: `flutter analyze`
- Code generation (if used): `dart run build_runner build --delete-conflicting-outputs`
- Localization generation: `flutter gen-l10n`
- Outdated dependencies: `flutter pub outdated`

## Project structure

```
lib/
  main.dart            app entrypoint — runApp and top-level wiring only
  app.dart             root widget, theme, routing, localization setup
  src/                 all application code (private to this app)
    features/          one folder per feature, each with:
                         data/          repositories, data sources, DTOs
                         domain/        entities and business rules
                         presentation/  screens, widgets, state management
    services/          abstract contracts to external services and the
                         platform, plus per-platform implementations
                         (replaces "utilities" from the Go layout)
    core/              shared theme, routing, constants, errors, extensions
    shared/            reusable widgets and helpers used by 2+ features
  l10n/                .arb localization files
packages/              local Dart/Flutter packages meant to be imported by
                         other modules (the equivalent of "pkg")
test/                  unit and widget tests, mirroring the lib/ layout
integration_test/      integration / end-to-end tests that run the built app
assets/                images, fonts, and other bundled assets
android/ ios/ web/ macos/ windows/ linux/   platform runners
.github/               workflows, issue templates, PR template
pubspec.yaml
analysis_options.yaml
CHANGELOG.md
```

- Unit and widget tests live under `test/` mirroring the path of the code they test (`lib/src/features/auth/data/auth_repository.dart` is tested by `test/src/features/auth/data/auth_repository_test.dart`), per Flutter convention.
- `integration_test/` is reserved for integration/end-to-end tests that exercise the real app on a device, emulator, or browser.
- New application code goes under `lib/src/`. Code that is meant to be imported by other modules or apps goes in its own package under `packages/`.
- Additional features must be created in their own folder under `lib/src/features/`.
- Do not import another package's or feature's `src/` internals; go through its public library file.

## Testing expectations

- Every new piece of logic gets a unit test; every new screen or reusable widget gets a widget test; every user-facing flow gets an integration test when it crosses screens or platform services.
- Platform-specific implementations are tested through their interface with fakes, and run on at least one real target for the platform they serve.
- A "full suite" run is `dart format --set-exit-if-changed .`, `flutter analyze`, `flutter test`, and `flutter test integration_test` on at least one mobile target and web (or the narrower set of targets the task affects).
- Golden tests are generated on one pinned platform only, to avoid cross-platform rendering drift; do not regenerate goldens unless the visual change is intentional and called out in the PR.
