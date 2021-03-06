# Konstraint

[![Go Report Card](https://goreportcard.com/badge/github.com/plexsystems/konstraint)](https://goreportcard.com/report/github.com/plexsystems/konstraint)

![logo](images/logo.png)

Konstraint is a CLI tool to assist with the creation and management of constraints when using [Gatekeeper](https://github.com/open-policy-agent/gatekeeper).

## Why this tool exists

When writing policies for Gatekeeper, the Rego must be added to [ConstraintTemplates](https://github.com/open-policy-agent/gatekeeper#constraint-templates) in order for Gatekeeper to enforce the policy. This creates a scenario in which the Rego is written in a `.rego` file, and then copied into the ConstraintTemplate. When a change is needed to be made to the Rego, both instances must be updated.

Gatekeeper also supports importing _libraries_ into `ConstraintTemplates` by leveraging the `libs` field. If a change is required in the imported library, each template must be updated to include this new change.

Additionally, since policies are evaluated in the context of `AdmissionReviews`, the `input` used when evaluating the policy is different than if the policy was being evaluated against a plain `.yaml` file with [Conftest](https://github.com/open-policy-agent/conftest).

`Konstraint` aims to:

- Auto-generate both `ConstraintTemplates` and the Constraints themselves. Your `.rego` files are the source of truth.

- Enable users to use the same policies for local development with `.yaml` and on a Kubernetes cluster with Gatekeeper. No need to write two different sets of policies! This is accomplished with the _Kubernetes library_.

### Kubernetes library

In the [examples/lib](examples/lib) directory, there is a `kubernetes.rego` file that enables policies to be written for both Conftest and Gatekeeper.

#### Purpose

When Gatekeeper receives an AdmissionReview, the input document in OPA will be of the form `input.review.object.kind`. However, when validating the manifests locally as `yaml` files, the input will just be `input.kind`. This makes it difficult to use the same policy for both solutions.

By first validating the Kubernetes manifests with `Conftest` on a local machine, we are able to catch manifests that would otherwise violate policy without needing to deploy to a cluster running Gatekeeper.

## Installation

```text
GO111MODULE=on go get github.com/plexsystems/konstraint
```

## Usage

Create `ConstraintTemplates` and Constraints from `Rego` policies:

`konstraint create <dir>`

This will generate both _templates_ (template.yaml) and _constraints_ (constraint.yaml) for the policies found in the directory (and subdirectories), ignored test files (`*_test.rego`).

### Flags

`--ignore` Ignores all occurances of a given directory by name.
_Example: konstraint create . --ignore combined-policies_

This can be useful if you are also using [Conftest](https://github.com/open-policy-agent/conftest) and have a separate directory for policies intended to be used with the [--combine](https://www.conftest.dev/options/#-combine) flag, and not added to a Kubernetes cluster.

## Template and Constraint Naming

The name of the ConstraintTemplate is derived from the name of the folder that the policy was found in.

For example, a policy found in: `policies/pod-volume-size-limits/src.rego` generates the following in the `policies/pod-volume-size-limits` directory:

- A `ConstraintTemplate` (as `template.yaml`) with the name `podvolumesizelimits` and a `Kind` value in the spec section to `PodVolumeSizeLimits`.

- A _constraint_ of `kind: PodVolumeSizeLimits` and name `podvolumesizelimits` (as `constraint.yaml`)

The tool works best with a folder structure similar to how Gatekeeper itself structures policies and templates. [https://github.com/open-policy-agent/gatekeeper/tree/master/library](https://github.com/open-policy-agent/gatekeeper/tree/master/library)

## Importing Libraries

Importing a library is also supported, a rego library should be placed in the `lib` folder.

`Konstraint` will then add the Rego from the library into the `libs` section of the `ConstraintTemplate`.

## Future plans

### Set Constraint matchers and documentation based on header comments

Each `violation[msg]` rule defines a policy, the return `msg` being a message that describes how the policy was violated.

To further promote that the `.rego` file is the source of truth for policy, a form of block comments on each `violation` rule could be added that includes a human readible description of what the policy does. This comment block could also be used to set the matchers on the `Constraint` itself.

#### Proposal

```rego
# All images deployed to the cluster must not contain a latest tag.
# @Kinds Pod, DaemonSet, Deployment, StatefulSet
violation[msg] {
  has_latest_tag

  msg := k8s.format(sprintf("(%s) %s: Images must not use the latest tag", [k8s.kind, k8s.name]))
}
```

The human-readable content could then be extracted for automatic documentation generation, similar to [promdoc](https://github.com/plexsystems/promdoc).

The values after the `@Kinds` text could be used to set the `matchers` on the constraint. `Konstraint` could keep a mapping of the `apiGroup` for each `kind`:

```yaml
spec:
  match:
    kinds:
      - apiGroups: ["", "apps"]
        kinds: ["Pod", "DaemonSet", "Deployment", "StatefulSet"]
```
