# Structure

- .kiro/                    Kiro specs, steering and hooks
- skeleton/                 Shared template: config schema, validation,
                            readiness score, forms, QR passport page
- apps/garment-passport/    App 1. Contains only its config, sample
                            data and a thin app shell
- apps/software-passport/   App 2. Same shape as app 1
- examples/                 Optional extra examples

## Rules
- Apps import from skeleton. Skeleton never imports from apps.
- Industry-specific words appear only in an app's config, never in
  skeleton code.
- Each app must run on its own.