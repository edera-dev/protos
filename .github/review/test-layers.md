# What checks this repo has, and what runs on a PR

There are no tests in this repository. There is no code to run. The checks are
`buf lint` and `buf breaking`, and the second one is partly switched off for the
package that matters. Knowing exactly which rules are disabled is most of the
job.

## buf lint

Configured in `buf.yaml` under `lint`. The `STANDARD` category, with
`ENUM_ZERO_VALUE_SUFFIX` set to `_UNKNOWN` and three rules relaxed for the
the existing schema files: `ENUM_ZERO_VALUE_SUFFIX`,
`RPC_REQUEST_STANDARD_NAME` and `RPC_RESPONSE_STANDARD_NAME`. Anything lint
catches is out of scope for a review.

## buf breaking

Configured in `buf.yaml` under `breaking`, using the `WIRE_JSON` and `FILE`
categories. The important part is `ignore_only`, which disables these rules for
the path the live schema sits under:

- `FIELD_NO_DELETE`
- `ENUM_VALUE_NO_DELETE`
- `FIELD_SAME_NAME`
- `FIELD_SAME_JSON_NAME`
- `EXTENSION_NO_DELETE`

That is the live package. Removing a field, removing an enum value, renaming a
field, or changing a JSON name in it will pass CI. Do not tell an author that
buf covers a change without checking this list first.

## The workflow

`.github/workflows/buf-ci.yaml` runs on push, pull request and delete. It runs
the buf action, which performs lint, breaking and the registry push, and it
skips the whole step when the actor is dependabot, because the registry token is
not available in that context.

## The review checks themselves

`.github/workflows/pr-review.yml` runs the two advisory review checks,
including this one, through the shared workflow in `edera-dev/actions`. They
build, lint and test nothing this repository ships. Never count them as
coverage for a change.

## What is not checked anywhere

- A field that keeps its number, name and type and changes meaning.
- A new field that the receiver treats as required.
- An assumed upgrade order between two peers.
- A comment that no longer describes the field it documents.

For all four the schema itself is the only record, so a gap in the comment is a
real gap. This is the repository where "the documentation is the test" is
literally true.

## Where a gap usually is

- A removal with no `reserved` entry, so the number or name can be taken again.
- A rule in `ignore_only` that is hiding exactly the change under review.
- A semantic change with no textual record of the new meaning.
- A new field whose optionality in practice is not stated anywhere.
