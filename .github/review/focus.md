# Review focus

What the advisory review checks look for in this repository. The shared
workflow in `edera-dev/actions` supplies the review method; this file supplies
everything specific to this repository, and `test-layers.md` beside it says
where checks live and what runs on a pull request.

Each section starts at its `<!-- focus: NAME -->` line and runs to the next
one. The templates under `advisory-review/templates/` in `edera-dev/actions`
fix the names and show where each section lands. `FORK_SCOPE` is optional;
every other section is required, and a name no template uses fails the run.

<!-- focus: INTRO -->
This repository is a Protocol Buffers schema published to a registry and generated into more than one language. There is no runtime here to crash. The failure mode is entirely different: a schema change compiles everywhere, every generated client builds, and two peers built against different revisions then disagree about what the bytes on the wire mean. Review for what happens during a rolling upgrade, not for what happens at build time.

<!-- focus: SERIOUS -->
## 1. Serious defects

Almost every serious defect here is a wire compatibility problem, and the thing that makes them serious is that nothing fails at build time. Read the change as a peer that has not been updated would see it.

**A field number reused or repurposed.** A number that once meant something else, now attached to a different field, makes an old peer decode new data as the old type. When the wire types happen to be compatible the decode succeeds and produces a wrong value, which is worse than an error. Check the file's history for the number, not just its current state, and check `reserved` covers anything removed.

**A field or enum value removed.** Removing a field does not stop an old peer from sending it, and it does not stop a new peer from receiving it — it stops it from being named. Removing an enum value turns a value a peer still sends into the unknown case, which is only safe if every consumer handles the unknown case deliberately. **`buf breaking` will not catch either of these for the package in active use**: `FIELD_NO_DELETE`, `ENUM_VALUE_NO_DELETE`, `FIELD_SAME_NAME`, `FIELD_SAME_JSON_NAME` and `EXTENSION_NO_DELETE` are all listed under `breaking.ignore_only` in `buf.yaml` for the path the live schema sits under. Read that block before assuming a rule applies. The automated gate is off exactly where the schema is, so this is the review's job.

**A field renamed.** The number is what goes on the wire, so a rename is compatible at the binary level and breaks JSON encoding, generated accessor names, and every caller in every language that referenced the old name. Say which encoding and which consumers.

**A semantic change with no schema change.** The same field, same number, same type, now meaning something else: units changed, a value that used to be absolute is now relative, a field that was optional in practice becomes required in practice, an empty value that used to mean "unset" now means "cleared". No tool detects this. It is the change most worth catching in review because nothing else can.

**A required-in-practice field added.** A new field that the receiver treats as mandatory fails every message from a peer that does not know about it yet. New fields have to be optional in behaviour, whatever the syntax says, or the upgrade order becomes a constraint nobody wrote down.

**A oneof changed.** Moving an existing field into a oneof, or out of one, changes both the generated API and the runtime behaviour when more than one member is set. Adding a new member to an existing oneof is usually fine; check what the receiver does with a member it does not recognise.

**A default that peers read differently.** A change to what an unset or zero value means is a wire-compatible change with a runtime consequence. Enum zero values are the common case: the convention in `buf.yaml` is an `_UNKNOWN` suffix, and an enum whose zero value means something real gives every old peer that meaning by omission.

**A service or method signature changed.** Renaming an RPC, changing its request or response type, or changing streaming-ness breaks callers at the transport layer. Removing a method breaks them at runtime rather than at build time if they were generated earlier.

**Comments that are the contract.** In a schema repository the comments are the specification: units, ranges, nullability, ordering guarantees, and what an empty value means. A comment that is now wrong is a real finding here, not a style note, because there is no implementation in this repository to check it against.

<!-- focus: SUPPLY -->
## 2. Supply chain

Real, but rarely "peers disagree about the wire" serious — label these **Supply chain** so severity reads honestly.

Unpinned action refs in `buf-ci.yaml`, which holds the registry token. A change to `buf.yaml` that adds a path to `ignore_only`, which is not a dependency change but has the same effect: it turns off a check that was catching something. Any addition to `ignore_only` should say what it is for and why the compatibility it disables is acceptable.

A dependency added to `buf.lock`, or a registry module pulled in without obvious need.

<!-- focus: SKIPPED_TEST_FORMS -->
A rule added to `lint.ignore_only` or `breaking.ignore_only` in `buf.yaml`, a path excluded from a module, or a step made conditional so it stops running.

<!-- focus: SUPPRESSION_FORMS -->
A `buf:lint:ignore` comment directive, a rule moved into `ignore_only`, or a category downgraded.

<!-- focus: RIGHT_LEVEL -->
There is no test layer in this repository. The equivalent question is whether `buf breaking` covers the change, and for the path listed under `ignore_only` the answer is often no.

<!-- focus: NO_TEST_LAYER -->
That is the normal case here: this repository has no tests at all, and for the one package in active use several breaking-change rules are disabled. When the only thing that would have caught a change is a rule that is switched off, say that — it is more useful than asking for a test.

<!-- focus: OUT_OF_SCOPE -->
Style, naming and formatting, or anything `buf lint` catches — it runs on every PR, and the rules that are deliberately relaxed are recorded in `buf.yaml`.

<!-- focus: CALIBRATION_COST -->
a field number reused across a release boundary silently reinterprets data that a peer already sent, and that costs a lot more.

<!-- focus: CANNOT_CHECK_EXAMPLE -->
I could not see which generated clients are in use at which revision, so I am reading the change against the schema alone

<!-- focus: IMPLICATION_EXAMPLE -->
"field 7 was a string and is now a message with the same number, so a peer on the old schema decodes the new bytes as a corrupt string rather than failing" does.

<!-- focus: SERIOUS_DEFINITION -->
a change that makes two peers built against different revisions disagree about the bytes on the wire, a field number reused or repurposed, a removal that a still-running peer will keep sending, a semantic change that no tool can detect, or a new field that a receiver treats as required

<!-- focus: SAY_WHAT_HAPPENS -->
an old peer decodes the new field as the previous one; the JSON name changes and every JSON client stops finding the field; a peer that has not upgraded has every message rejected; the unknown enum value is treated as the zero value

<!-- focus: WRITE_BAD -->
The `state` field is declared as field 4 in `control.proto`, and field 4 previously held `legacy_state` according to the file's history, which was removed in an earlier commit without a `reserved` entry, so the new declaration reuses...

<!-- focus: WRITE_GOOD -->
A peer on the current schema will decode the new `state` field as the old `legacy_state` and get a wrong value rather than an error. Field 4 held `legacy_state` until it was removed without a `reserved` entry, and this change reuses the number for a field whose wire type happens to be compatible. Use the next free number and add `reserved 4;` so the old number can never be taken again.

<!-- focus: CLEAN_EXAMPLE -->
One new optional field with the next free number, and the comment says what an empty value means. Nothing concerning.

<!-- focus: OUTPUT_EXAMPLE -->
One problem I think should be fixed before merge: the field number is reused.

**Serious: field 4 is reused, so old peers will decode the new value as the previous field.**

A peer that has not been updated reads the new `state` as the `legacy_state` that used to occupy field 4 and gets a wrong value rather than an error, because the wire types are compatible. The new field in `control.proto` takes the number; the earlier field was removed without a `reserved` entry, so nothing prevented it.

Use the next free number and add `reserved 4;` alongside it. `buf breaking` will not catch this — `FIELD_NO_DELETE` is in `ignore_only` for this path.

<!-- focus: UNKNOWN_EXAMPLE -->
The new enum's zero value is named without the `_UNKNOWN` suffix the lint configuration expects elsewhere, and the rule is relaxed for this file. I could not tell from the schema alone whether any consumer relies on the zero value meaning something specific. No change requested.

<!-- focus: IMPACT_WORKED_EXAMPLE -->
The field number is reused, so a peer still on the previous revision decodes the new value as the field that used to hold that number. The wire types are compatible, so the decode succeeds and the peer acts on a wrong value instead of reporting an error.

<!-- focus: HOW_WRONG -->
- What does a peer built against the previous revision do with a message produced after this change, and what does a peer built against this revision do with a message produced before it? Answer both directions.
- Is any field number, JSON name, or enum value being reused, removed, or renamed?
- Does any field keep its number and type but change meaning — units, sign, relative versus absolute, what an empty value means?
- Does a new field have to be present for the receiver to work? If so, every peer that has not upgraded fails.
- Does the change assume an upgrade order between two sides? If it does, that order is a constraint nobody has written down.
- Would `buf breaking` catch this, given the `ignore_only` entries in `buf.yaml`?

<!-- focus: WHERE_TO_LOOK -->
- `buf.yaml`, specifically the `breaking.ignore_only` block. Several rules are disabled for the path the package in active use sits under, so "buf will catch it" is frequently wrong. Read the list before assuming a rule applies;
- `.github/workflows/buf-ci.yaml`, which runs lint, breaking and the registry push, and skips entirely for dependabot;
- the comments in the changed `.proto` file, which in this repository are the specification — a comment that contradicts the field is a real gap;
- the file's own history, for what a reused field number used to mean.

There are no tests in this repository. Do not look for a test file.

<!-- focus: PROPORTIONATE -->
Proposing a test suite for a schema repository is not proportionate. Proposing a `reserved` entry, a comment that states the contract, or removing an `ignore_only` line that is hiding a real check is.

<!-- focus: CONDITIONAL_EXAMPLE -->
"During a rolling upgrade, a peer still on the previous revision decodes field 4 as the field that used to hold that number and acts on a wrong value" names the condition and the result. "This could cause compatibility problems" names neither.

<!-- focus: TWO_SHAPES -->
- **The check that would catch it is switched off.** `breaking.ignore_only` in `buf.yaml` disables field deletion, enum value deletion, and both name rules for the path the live schema sits under. A change that any of those would have caught passes CI. Naming the specific rule and the specific path is the most useful thing this review can do.
- **No tool can catch it at all.** A field that keeps its number, name and type but changes meaning is invisible to every checker. The only protection is the comment that states the contract, so for this class the gap is a documentation gap and it is a real one.

<!-- focus: SMALLEST_LAYER -->
Pick the smallest thing that would prevent the failure. A `reserved` entry, for a removed number or name. A comment stating units, range, or what empty means, for a semantic change. Removing an `ignore_only` line, where the rule would now pass and is worth keeping on. A new field number rather than a reused one. Do not propose a test harness.

<!-- focus: COVER_BAD -->
buf breaking covers this. The workflow runs the WIRE_JSON and FILE categories against the registry revision, and the change only adds a field, which is compatible under both, and lint passes as well...

<!-- focus: COVER_GOOD -->
Adding a field with the next free number is compatible in both directions, and the comment states what an empty value means.

<!-- focus: CLEAN_NOTHING -->
Nothing here needs a check. It's a comment fix.

<!-- focus: GAP_EXAMPLE -->
One gap, and I'd close it with this PR since it is one line.

**Nothing stops field 4 from being taken again.**

The field was removed in this change without a `reserved` entry, so a later schema change can reuse the number and old peers will decode the new value as this one. `buf breaking` will not object: `FIELD_NO_DELETE` is in `ignore_only` for this path in `buf.yaml`.

Adding `reserved 4;` and `reserved "legacy_state";` to the message makes the number and the name permanently unusable, which is the assertion that protects against it.
