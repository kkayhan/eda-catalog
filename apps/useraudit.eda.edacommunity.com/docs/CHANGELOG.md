# Changelog

All notable changes to the EDA User Audit app are documented here.

Versions follow the EDA release the app is built and validated against, with
an incrementing build suffix: `v<eda-release>-<n>` (e.g. `v26.4.3-1`,
`v26.4.3-2`). When the target EDA release changes, the suffix restarts at `-1`.

## v26.4.3-1

- Adopt the EDA-release-tied version scheme. This build targets and is
  validated against EDA **26.4.3**. Application code is unchanged from v0.8.0;
  this release re-baselines the version number to track the EDA release.
- Fix stale controller `VERSION` string (was `v0.7.1`; now matches the
  release tag).
- Bump project `builderVersion` to `v26.4.3` to match the toolbox.

## v0.8.0 and earlier

Pre-date the EDA-release-tied scheme. See git tags
`apps/useraudit.eda.edacommunity.com/v0.8.0`, `.../v0.7.1`, `.../v0.7.0`.
