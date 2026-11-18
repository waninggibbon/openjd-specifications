* Feature Name: Step Approvals
* Author(s): [folouiseAWS](https://github.com/folouiseAWS), Nate MacKinlay <[waninggibbon](https://github.com/waninggibbon)>
* RFC Tracking Issue: https://github.com/OpenJobDescription/openjd-specifications/issues/200
* Start Date: 2026-11-18
* Specification Version: 2023-09 extension STEP_APPROVALS
* Accepted On: (pending)
* Depends On:
  * RFC 0002, Model Extensions (https://github.com/OpenJobDescription/openjd-specifications/issues/57)

## Summary

This RFC adds an optional `approval` property to `<StepTemplate>`, behind a new `STEP_APPROVALS`
extension. With it, a person's decision becomes a condition for scheduling a Step. When the Step would
first become eligible to run, the render management system opens an approval request. The request
shows a message and links written by the template author. The Step's Tasks may be scheduled only while
an approval is in effect, and no Worker host is held while the request waits. A rejection or expiry
means the Step does not complete successfully, so the Steps that depend on it never run. A Step with an
`approval` and no `script` is an *approval-only Step*: a review point in the Step graph that runs
nothing and completes when someone approves it.

The template says *that* a decision is needed and *what the approver sees*. The render management
system decides *who* may decide, and how they are notified.

## Basic Examples

### Review the preview before a costly final render

```yaml
specificationVersion: "jobtemplate-2023-09"
extensions: [ STEP_APPROVALS ]
name: "{{Param.ShotName}} render"
parameterDefinitions:
- name: ShotName
  type: STRING
- name: Frames
  type: STRING
  default: "1-240"
- name: OutputDir
  type: PATH
  objectType: DIRECTORY
  dataFlow: OUT
steps:
- name: PreviewFrames
  parameterSpace:
    taskParameterDefinitions:
    - name: Frame
      type: INT
      range: "{{Param.Frames}}:10"
  script:
    actions:
      onRun:
        command: render
        args: [ "--quality", "preview", "--frame", "{{Task.Param.Frame}}",
                "--output", "{{Param.OutputDir}}/preview" ]

- name: ReviewPreview
  dependencies:
  - dependsOn: PreviewFrames
  approval:
    message: |
      Check the preview frames of {{Param.ShotName}} before the final render of frames
      {{Param.Frames}} runs on hosts with four or more GPUs.
    links:
    - label: Preview frames
      parameter: OutputDir
      subpath: preview
    expiresAfterSeconds: 259200

- name: FinalFrames
  dependencies:
  - dependsOn: ReviewPreview
  hostRequirements:
    amounts:
    - name: amount.worker.gpu
      min: 4
  parameterSpace:
    taskParameterDefinitions:
    - name: Frame
      type: INT
      range: "{{Param.Frames}}"
  script:
    actions:
      onRun:
        command: render
        args: [ "--quality", "final", "--frame", "{{Task.Param.Frame}}",
                "--output", "{{Param.OutputDir}}/final" ]
```

`ReviewPreview` is an approval-only Step. It opens a request once `PreviewFrames` completes, and
`FinalFrames` waits for the decision. If someone requeues `PreviewFrames` after the review, the
approval stops being in effect, and the new preview frames have to be reviewed again.

### Approve a Step before it starts

An `approval` may also sit on a Step that has a `script`. Here the Job's only Step waits for someone
to confirm a license before it runs:

```yaml
steps:
- name: UseLicensedAsset
  approval:
    message: "Confirm that the license for {{Param.AssetName}} covers this use."
    links:
    - label: License record
      url: "https://licenses.example.com/assets/{{Param.AssetId}}"
  script:
    actions:
      onRun:
        command: process-asset
        args: [ "{{Param.AssetName}}" ]
```

## Motivation

Many pipelines have a point where a person, not a program, decides whether work continues:

* A supervisor checks preview frames before the full-quality render runs on large GPU hosts.
* Someone checks a sample of prepared training data before training starts.
* A production confirms a third-party asset's license before a Job uses it.
* A budget owner allows a Job that is expected to cost more than a threshold.

Open Job Description has no way to express this today, and each workaround has a cost:

* **A gate Job Parameter checked by a script.** The submitter approves their own work by typing a
  value. Nothing records who decided or when. A Worker host is assigned only for the Task to fail, and
  a decision made later means resubmitting the Job.
* **Separate Jobs for each part of the pipeline**, with someone submitting or resuming the next Job by
  hand. The decision point is not visible in any template, and the Jobs' dependency on each other
  lives in a person's head.
* **A Task that polls an external system for an answer.** It holds a Worker host, and often an
  expensive one, for the whole wait. It also needs network access and credentials for the external
  system.

Each of these puts a person's decision inside the Job, where the Job itself can make or fake it. The
decision should sit outside the Job, in the render management system, where it can be controlled
and audited.

### Use cases

1. **Review gate:** an approval-only Step between a cheap Step and an expensive one. This is the
   most common case.
2. **Authorize before starting:** an `approval` on a root Step with a `script`. Use it for license,
   budget or compliance checks before any work runs.
3. **Hand-off between teams:** an approval-only Step at the boundary between departments, for example
   layout to lighting. A render management system can let a review tool, such as a production
   tracker, record the decision through its own integration.

### Backward compatibility

* Templates that don't list `STEP_APPROVALS` are unaffected. Without the extension, `script` stays
  required and `approval` is not allowed.
* An implementation that doesn't support `STEP_APPROVALS` rejects templates that list it, as RFC 0002
  requires. This fails safely: the gate is never silently ignored.
* Sessions and Worker hosts don't change. `approval` is handled entirely by the render management
  system, and a Worker host never needs to understand it.
* The meaning of "fully completed successfully" for a Step without an `approval` does not change.

## Specification

The wiki changes in this pull request are the normative text. Where this RFC and the wiki differ, the
wiki governs.

### Terminology

* **Approval request** — The render management system's record of whether a person has allowed a
  Step's Tasks to be scheduled. A Step has at most one current request. Earlier requests are kept for
  the audit trail.
* **Decision** — An approver's answer to a request: approve or reject.
* **In effect** — An approval is in effect when the Step's current request is approved.
* **Approval-only Step** — A Step with an `approval` and no `script`.

### Schema modifications ([Template Schemas §3])

* `<StepTemplate>` gains an optional `approval: <StepApproval>` property, available with
  `STEP_APPROVALS`.
* `script` becomes optional, but only when `approval` is present. A Step with neither stays invalid.
  With `FEATURE_BUNDLE_1`, a Step-level simple action (`bash:`, `python:`, and so on) counts as a
  `script`, so an approval-only Step has neither.
* An approval-only Step must not have `stepEnvironments`, `hostRequirements`, `parameterSpace` or
  `let`. It runs nothing, so these would have no meaning. This restriction can be relaxed later
  without breaking any template.
* [§3.2 `<StepDependency>`][Template Schemas §3.2] gains a note: for a depended-upon Step that has an
  `approval`, "fully completed successfully" requires an approval in effect.

### `<StepApproval>` ([Template Schemas §3.7])

```yaml
message: <ApprovalMessage> # @fmtstring
links: [ <ApprovalLink>, ... ] # @optional
expiresAfterSeconds: <integer> # @optional
onReject: "FAIL" # @optional
```

* **`message`** is required. It is what the approver reads, so it should say what to check and what
  approving allows.
* **`links`** point the approver at what to review. Each `<ApprovalLink>` has a `label` and exactly one
  target, written in one of two ways:
  * a `url`, which must resolve to an absolute `http` or `https` URI; or
  * a `parameter`, naming a PATH-type Job Parameter, with an optional relative `subpath`.

  PATH values are shown as the submitter gave them, without path mapping, because the approver's host
  has nothing to do with any Worker host.
* **`expiresAfterSeconds`** sets how long a request may wait for a decision. It must be between 60 and
  2,592,000 (30 days).
* **`onReject`** accepts only `FAIL`, which is also the default. It is reserved so that a later
  revision can add other outcomes without a new extension (see [Future Work](#future-work)).

The format strings in `<StepApproval>` are resolved when the Job is created, not on a Worker host.
Their scope is `RawParam.*`, `Param.*` for parameters that are not PATH type, and `Job.Name` and
`Step.Name` (with `EXPR`). The value of a PATH parameter is available only as `RawParam.*`. The same
holds for every other format string resolved when the Job is created, because path mapping happens
only on a Worker host.

### Scheduling semantics ([How Jobs Are Run § Step Approvals])

**Lifecycle.** An approval request is in one of four states:

| State      | Tasks may be scheduled |
| ---------- | ---------------------- |
| `WAITING`  | No                     |
| `APPROVED` | Yes                    |
| `REJECTED` | No                     |
| `EXPIRED`  | No                     |

**Opening.** A request opens when the Step would first become eligible for scheduling. That is when
all its dependencies have fully completed successfully, or when the Job becomes eligible to run if the
Step has no dependencies. It never opens earlier, so the approver can review what the dependencies
produced.

**Gating.** A Step's Tasks may be scheduled only while an approval is in effect. Waiting is a
scheduling condition, like a dependency that is not yet resolved. No Session runs and no Worker host
is held while a request waits.

**No bypass.** The gate applies to every way the render management system can make Tasks eligible for
scheduling. That includes retrying, requeuing or resuming the Step's Tasks, and requeuing or
restarting the whole Job.

**Expiry.** The expiry clock starts when the request opens. Expiry never approves.

**Rejection and expiry.** A Step whose request is `REJECTED` or `EXPIRED` does not fully complete
successfully. None of its Tasks run, and the Steps that depend on it are never scheduled.
Implementations should show this differently from a Step whose Tasks failed. Retrying or requeuing a
rejected or expired Step opens a new request.

**Approval-only Steps.** An approval-only Step fully completes successfully when it is approved, and
not before. Having nothing to run does not make it complete.

**Re-approval when a dependency re-runs.** An approval applies to the outputs that the Step's
dependencies had when the decision was made.

* If a direct dependency stops being fully completed successfully after the decision, for example
  because its Tasks are requeued, the approval is no longer in effect.
* Tasks of the gated Step that have not started stay unscheduled. Once the dependencies complete
  again, a new request opens with a fresh expiry clock.
* Tasks that already succeeded are left alone. Whether running Tasks finish or are interrupted is up
  to the implementation.
* Requeuing the approved Step itself, while its dependencies stay complete, keeps the approval in
  effect.

**Where a decision may come from.**

* Only principals that the render management system authorizes can make a decision, through its own
  mechanism.
* It must not accept a decision made with any credential that it makes available to the Job, its
  Sessions or its Worker hosts.
* A decision cannot come from Job Parameters, Task output or any other content of the Job.
* The render management system records who decided, when, and any comment the approver gave.

**Conformance.** An implementation that accepts `STEP_APPROVALS` must enforce the gate. It must not
accept the extension and then ignore `approval`.

### Local runs

`openjd run` is a local, single-user runner, so it implements the gate by asking at the terminal:

* With a terminal, it prompts at the gate, showing the resolved message and links, and waits for an
  answer.
* `--approve-step <StepName>` approves that gate in advance, and may be given more than once.
* With no terminal and no `--approve-step` for the gate, the gate counts as rejected. A
  non-interactive run therefore never approves anything.

This is runner behavior, not normative specification text.

[Template Schemas §3]: https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#3-steptemplate
[Template Schemas §3.2]: https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#32-stepdependency
[Template Schemas §3.7]: https://github.com/OpenJobDescription/openjd-specifications/wiki/2023-09-Template-Schemas#37-stepapproval
[How Jobs Are Run § Step Approvals]: https://github.com/OpenJobDescription/openjd-specifications/wiki/How-Jobs-Are-Run#step-approvals

## Design Choice Rationale

### A property on the Step, not a `type: APPROVAL` Step

`<StepTemplate>` has no type field today. Adding one would change the schema's structure for every
implementation, just to tell two kinds of Step apart. A property covers both use cases with one
construct: an approval-only Step for a review between Steps, and an approval on a Step with a
`script` for authorizing work before it starts. An approval-only Step still gets everything an
ordinary Step has in the graph: a name, dependencies, and a status that its dependents wait on.

### The gate sits before the Step runs, not after it completes

The alternative, "this Step is not complete until approved", also ties the approval to the reviewed
output. But then a Step whose Tasks all succeeded would not count as complete. Every consumer of Step
status, including status rollups, UIs and dependency resolution, would have to learn that. Putting
the gate before the Step keeps the existing meaning of a dependency, and an approval-only Step covers
the case of approving a Step's output.

### Waiting is a scheduling condition

Render management systems already hold Tasks back until their dependencies resolve, without starting
a Session. Treating an approval as one more such condition keeps a waiting request from holding a
Worker host. It also lets implementations reuse the way they already hold and release Tasks.

### The no-bypass rule names a principle, not a list of operations

Render management systems offer many ways to make Tasks run again. If any one of them skipped the
gate, the gate would be worthless as a control. The rule therefore covers every way of making Tasks
eligible for scheduling, including operations that are added later.

### A dependency re-run requires a new approval

Without this rule, the following could happen (from [discussion #199]):

1. The approver approves the preview frames.
2. Someone requeues the preview Step, and it produces different frames.
3. The final render runs on frames nobody reviewed.

Ordinary dependencies don't promise that dependents wait again, but an approval is a judgement about
specific outputs. Only direct dependencies are watched. When a Step further up re-runs, the direct
dependencies become incomplete as a consequence, which already triggers the rule.

### An approval-only Step does not complete just because it has nothing to run

A Step with no work could easily be treated as already complete, and that would approve it
automatically. The specification says outright that such a Step completes only when approved.

### Rejection blocks dependents and does not skip them

Blocking is the conservative choice, and it matches what already happens when a dependency does not
complete successfully. Skipping dependents raises the same questions as #81 (`enableIf`), so it is
left for later. `onReject` reserves the place for it.

### The template names no approvers

Who may approve is a property of the organization, not of the work. The same template moves between
studios, queues and render management systems. Naming principals in the template would make it
specific to one system, and the submitter could pick their own approvers. The render management
system maps the request to approvers, for example through queue-level configuration.

### Decisions can't come from inside the Job

Every workaround in [Motivation](#motivation) fails in the same way: the Job, or its submitter, can
approve itself. The requirement is written in terms an implementation can test: reject decisions made
with credentials the render management system gives the Job. "The Job must not decide" on its own is
not testable.

### Format strings are resolved when the Job is created

The approver reads the message before any of the Step's Tasks run, often on a host that is not a
Worker host. So the message can only use values known when the Job is created. Showing PATH values as
submitted avoids picking a path mapping for an approver whose host has no mapping rules.

## Security Considerations

### Links are content from the template author

Approval links are written by the template author and shown to someone who is likely to trust them.
A template could use them for phishing. The schema allows only `http` and `https` URLs. Render
management systems should show where a link really goes rather than only its label, and should treat
`message` as untrusted text, not as markup.

### Notifications and confidentiality

Approval messages often contain project names or other confidential details. That matters most when
render management systems send them through third-party channels, such as chat or mobile push.
Implementations should be able to send a generic notice and show the details only after the approver
signs in.

### Approving from inside a message

Approve buttons in a chat message, or one-click links in an email, make the decision with the
notification system's credentials, not the approver's. They also act for whoever ends up with the
message, including someone it was forwarded to. Implementations should make the decision only after
the approver has authenticated as themselves.

## Prior Art

* **GitHub Actions environments with required reviewers.** A job that targets a protected environment
  waits for an allowed reviewer, and holds no runner while it waits. The workflow names the
  environment, and the repository settings name the reviewers. This is the same split as this RFC:
  the template says *that* approval is needed, and the system says *who*.
* **AWS CodePipeline manual approval actions.** A pipeline stage pauses for an approval. It can show a
  review URL and comments, sends notifications, and expires after 7 days if nobody decides. This is
  the basis for the `message`, `links` and expiry fields.
* **Argo Workflows `suspend` templates.** A workflow node pauses until it is resumed or a duration
  passes. Argo has no approver identity and no rejection: resuming continues the workflow.
* **Jenkins `input` step.** It pauses a pipeline for a decision and can restrict who may submit it.
  In declarative pipelines it holds an executor unless written outside an agent, which is a known
  pitfall. This RFC avoids it by making waiting a scheduling condition.
* **Production trackers (ShotGrid/Flow, ftrack).** Review statuses gate the next department's work,
  but outside the render manager. An approval-only Step lets the render manager hold the gate, while
  an integration can still record the decision.

## Prototype

No prototype in [openjd-rs] yet. A prototype in openjd-model-for-python and openjd-cli, and an
implementation in a render management system, are planned alongside this RFC. This section will
record what they teach us.

[openjd-rs]: https://github.com/OpenJobDescription/openjd-rs

## Implementation Impact

- **Python (openjd-model)**:
  - Schema: the `STEP_APPROVALS` extension, the `approval` model, `script` made optional, and the
    approval-only constraints.
  - Resolving the `message` and `links` format strings when the Job is created.
  - Small, self-contained changes.
- **Rust (openjd-rs)**: the same as Python.
- **openjd-sessions**: no change. `approval` is never part of a Session.
- **openjd-cli**: the gate in `openjd run` (the prompt, `--approve-step`, and treating a run with no
  terminal as rejected), plus showing approvals in `openjd check` and `openjd summary`.
- **Render management systems**: most of the work is here.
  - Storing approval requests and decisions.
  - Applying the gate to every way of making Tasks eligible.
  - Re-arming the gate when a dependency is requeued.
  - Running expiry.
  - Authenticating decision makers and rejecting the Job's own credentials.
  - Notifications.

  An approval-only Step may need a placeholder Task that is never dispatched, if the system cannot
  represent a Step with no Tasks.

## Rejected Ideas

### A gate Job Parameter or a polling Task

These are the workarounds described in [Motivation](#motivation). Both put the decision inside the
Job.

### Approval on the dependency link

Here the `approval` would go on a `<StepDependency>`, as in "`FinalFrames` depends on `PreviewFrames`,
approved". This ties the approval to one upstream Step's output and avoids Steps with nothing to run.
It was rejected for two reasons:

* It can't gate a root Step, so the authorize-before-starting case is lost.
* If several Steps depend on one reviewed Step, each link opens its own request, so one review means
  several approvals.

### A general asynchronous Step type

This would be a Step that runs outside a Worker host, with `onStart`, `onProgress` and `onExit`
handlers, and could hand an approval to an external tool ([discussion #199]). It deserves its own RFC,
because running work without a Worker host raises questions about execution, credentials and
isolation that approvals don't. This RFC doesn't rule it out. A render management system can already
let external tools make approval decisions through its own API.

### Naming approvers in the template

See [The template names no approvers](#the-template-names-no-approvers).

## Open Questions

* **Routing hint (`audience`).** Should a template be able to give an opaque hint, such as
  `audience: budget`, that a render management system maps to a group of approvers? The fields of a
  released extension can't change without breaking validators, so this has to be decided before
  release. Leaving it out means a new extension later.
* **Suspended Jobs.** Does a request's expiry clock keep running while its Job is suspended?
* **`let` in approval-only Steps.** v1 disallows `let`. Should bindings be visible to `message` and
  `links`?

## Future Work

* **Skipping dependents on rejection**, sharing its semantics with #81. It would add values to
  `onReject`.
* **Approvals that carry a value into later Steps**, for example a chosen take. This builds on #111.
* **Multiple approvers or a quorum.** These are left to render management systems for now. The
  template could later state a minimum number of approvals.

[discussion #199]: https://github.com/OpenJobDescription/openjd-specifications/discussions/199

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
