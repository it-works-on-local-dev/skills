# Changelog

## [1.3.0](https://github.com/it-works-on-local-dev/skills/compare/v1.2.1...v1.3.0) (2026-10-25)


### Features

* ask whether to proceed, defer or decline findings after 4 review rounds ([#25](https://github.com/it-works-on-local-dev/skills/issues/25)) ([8a5d830](https://github.com/it-works-on-local-dev/skills/commit/8a5d830688f7aa2271b2075356ce987c5e69cae0))
* have beads-pr apply a requested review model, effort or credential ([#22](https://github.com/it-works-on-local-dev/skills/issues/22)) ([d8f6286](https://github.com/it-works-on-local-dev/skills/commit/d8f62869685f561ba044f9590ef08c8ac577eb47))
* ship a one-bead wave's once-per-wave files on its bead PR ([#33](https://github.com/it-works-on-local-dev/skills/issues/33)) ([e9bdf58](https://github.com/it-works-on-local-dev/skills/commit/e9bdf58b27d888231d10ff596303aae35b45c48d))


### Bug Fixes

* give create-pr and beads-pr the same thread replies and squash commits ([#26](https://github.com/it-works-on-local-dev/skills/issues/26)) ([39f9926](https://github.com/it-works-on-local-dev/skills/commit/39f9926f39b6166c0a322582079e5c8945b08ac9))
* have beads-pr handle missing, flaky and stuck CI the way create-pr does ([#28](https://github.com/it-works-on-local-dev/skills/issues/28)) ([26d7a39](https://github.com/it-works-on-local-dev/skills/commit/26d7a3917385155f0c2c018ce08ebf26cebb07f2))
* have beads-pr handle partial, stalled and repeated reviews the way create-pr does ([#27](https://github.com/it-works-on-local-dev/skills/issues/27)) ([6946954](https://github.com/it-works-on-local-dev/skills/commit/69469543b15e9c509db1d05ac195528c277e44bd))
* have beads-pr name the PR that broke main instead of offering a re-run ([#31](https://github.com/it-works-on-local-dev/skills/issues/31)) ([a4a0ede](https://github.com/it-works-on-local-dev/skills/commit/a4a0edec773b2e47c128cda412971b33cc3e8294))
* have beads-pr pick the CI run on main that covers the whole wave ([#29](https://github.com/it-works-on-local-dev/skills/issues/29)) ([73d90ea](https://github.com/it-works-on-local-dev/skills/commit/73d90ea171f362ee3878af9dadb1a6b38cb46586))
* have beads-pr triage name the PR that broke main instead of re-running its check ([#32](https://github.com/it-works-on-local-dev/skills/issues/32)) ([a89122d](https://github.com/it-works-on-local-dev/skills/commit/a89122d2bb69d38e8c93ef445b67bf3efc50f9a4))
* have setup list the Contents permission for the review app ([#23](https://github.com/it-works-on-local-dev/skills/issues/23)) ([6a562b3](https://github.com/it-works-on-local-dev/skills/commit/6a562b3c0540c38ff1141adb10d44d05a1118e17))
* say a PR description's /review line also picks the first review's credential ([#20](https://github.com/it-works-on-local-dev/skills/issues/20)) ([7e64b46](https://github.com/it-works-on-local-dev/skills/commit/7e64b46cc3206bbe4a97cfcda002465d0234daa1))
* say where a failed Claude review's reason shows ([#24](https://github.com/it-works-on-local-dev/skills/issues/24)) ([1b48193](https://github.com/it-works-on-local-dev/skills/commit/1b481934ab13b5f998255e48fc309d7d5814fa2c))
* stop /iwol:start renaming the session ([#30](https://github.com/it-works-on-local-dev/skills/issues/30)) ([f9f1e56](https://github.com/it-works-on-local-dev/skills/commit/f9f1e5679c3a0185a6ee50334be57b99e0a40cd6))

## [1.2.1](https://github.com/it-works-on-local-dev/skills/compare/v1.2.0...v1.2.1) (2026-10-23)


### Bug Fixes

* say how a requested review's failed runner pick shows ([#17](https://github.com/it-works-on-local-dev/skills/issues/17)) ([9363128](https://github.com/it-works-on-local-dev/skills/commit/9363128c6fae4b4a50f9db207606fbb88a24103c))

## [1.2.0](https://github.com/it-works-on-local-dev/skills/compare/v1.1.0...v1.2.0) (2026-10-22)


### Features

* have setup ask which Claude credential the review stub passes ([#14](https://github.com/it-works-on-local-dev/skills/issues/14)) ([164a8af](https://github.com/it-works-on-local-dev/skills/commit/164a8afce2dd8834756d22a11031a4b43569d523))

## [1.1.0](https://github.com/it-works-on-local-dev/skills/compare/v1.0.0...v1.1.0) (2026-10-22)


### Features

* ship once-per-wave generated files in one refresh PR per wave ([#13](https://github.com/it-works-on-local-dev/skills/issues/13)) ([d70cc29](https://github.com/it-works-on-local-dev/skills/commit/d70cc298afa5e8c00c3ff877ce4dcabbb204b78f))


### Bug Fixes

* have setup write a picker job that starts only when a review might follow ([#11](https://github.com/it-works-on-local-dev/skills/issues/11)) ([efd1d4f](https://github.com/it-works-on-local-dev/skills/commit/efd1d4f99346be955e60a440ffa8bb2297418829))

## 1.0.0 (2026-10-22)


### Features

* let create-release run without a deploy workflow or a production version URL ([#4](https://github.com/it-works-on-local-dev/skills/issues/4)) ([84997c3](https://github.com/it-works-on-local-dev/skills/commit/84997c317da57b67a9cc7ad20be54bdc607b2983))
* make setup ask which reviewer the repository uses and write the public review stub ([#2](https://github.com/it-works-on-local-dev/skills/issues/2)) ([13d393d](https://github.com/it-works-on-local-dev/skills/commit/13d393dea1bf0ccdf77f231ec1a67dbeeccb6a37))
* update the config template for the public review workflow, self-hosted CI and releases without a deploy ([#3](https://github.com/it-works-on-local-dev/skills/issues/3)) ([dbcbed4](https://github.com/it-works-on-local-dev/skills/commit/dbcbed4fb3a6fd8df5ee25261d2e6d861908148c))
