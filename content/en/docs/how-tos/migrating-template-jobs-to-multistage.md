---
title: "Migrating CI Jobs from Templates to Multi-stage Tests"
date: 2020-10-30T21:49:06+02:00
draft: false
---

OpenShift CI offers two mechanisms for setting up end-to-end tests: older 
template-based tests and newer [multi-stage test workflows](/docs/architecture/step-registry/).
Template-based tests cause problems for both DPTP and OpenShift CI users:
 - they need intensive support
 - backing implementation is buggy and notoriously difficult to maintain
 - jobs using templates are hard to extend, modify, and troubleshoot

These concerns were addressed in designing the multi-stage workflows, which
supersede template-based tests. DPTP wants to migrate all existing template-based
jobs to multi-stage workflows in the medium term.

## Migrating Jobs Generated from `ci-operator` Configurations

Most template-based jobs are generated from `ci-operator` configuration stanzas.
Migrating these jobs is easier and can be done almost mechanically. Soon, DPTP
will migrate all existing `ci-operator` configurations to multi-stage workflows
automatically.

### `openshift_installer_src`

The tests using the `openshift_installer_src` stanza install OpenShift using a
generic IPI installation workflow, execute a provided command in the context of
the `src` image (which contains a git clone of the tested repository), and
teardown the cluster. The template provides tests a writable `$HOME` and injects
the `oc` binary from the `cli` image, which must be explicitly imported in
`base_images`.

These jobs can be migrated to multi-stage using an `ipi-$PLATFORM` workflow
(like [`ipi-aws`](https://steps.ci.openshift.org/workflow/ipi-aws)) and replacing
its `test` stage with matching inline steps. The resulting configuration is
more verbose. This is a consequence of multi-stage tests being more flexible, 
allowing configuration of the elements that in templates were hardcoded.

#### Before (template-based)

```yaml
tests:
- as: e2e-test
  commands: make test
  openshift_installer_src:
    cluster_profile: aws
```

#### After (multi-stage)
```yaml
tests:
- as: e2e-test
  steps:
    cluster_profile: aws     # needs to match the workflow
    test:                    # override the `test` section of the workflow
    - as: test               # name of the step
      cli: latest            # inject the `oc` binary from specified release into image used by the step 
      commands: make test    # execute `make test`...
      from: src              # ...inside the src image
      resources:             # explicitly specify resources for the step
        requests:
          cpu: 100m
    workflow: ipi-aws        # `ipi-$PLATFORM` workflows implement the "usual" OpenShift installation
```

#### Gotchas

These are the possible problems you may encounter when porting an `openshift_installer_src`
test to a multi-stage workflow.

##### Hardcoded kubeadmin password location

Tests that use the file containing the kubeadmin password and hardcode its
location (`/tmp/artifacts/installer/auth/kubeadmin-password`) provided by
the template will not find the file at that location anymore.

**Resolution:** Port the test to use [`$KUBEADMIN_PASSWORD_FILE`](/docs/architecture/step-registry/#available-environment-variables) environmental variable instead.

### `openshift_installer_custom_test_image`

The tests using the `openshift_installer_custom_test_image` stanza install
OpenShift using a generic IPI installation workflow, execute a provided command
in the context of the specified image, and teardown the cluster. The template
provides tests a writable `$HOME` and injects the `oc` binary from the `cli`
image, which must be explicitly imported in `base_images`.

These jobs can be migrated to multi-stage in an almost identical way to
the [`openshift_installer_src`](#openshift_installer_src) ones by using an
`ipi-$PLATFORM` workflow (like [`ipi-aws`](https://steps.ci.openshift.org/workflow/ipi-aws))
and replacing its `test` stage with a matching inline step. The only difference
is that the specified image will be different.

```yaml
tests:
- as: e2e-test
  commands: make test
  openshift_installer_custom_test_image:
    cluster_profile: aws
    from: stable:aws-ebs-csi-driver-operator-test
```

#### After (multi-stage)
```yaml
tests:
- as: e2e-test
  steps:
    cluster_profile: aws                             # needs to match the workflow
    test:                                            # override the `test` section
    - as: test                                       # name of the step
      cli: latest                                    # inject the `oc` binary 
      commands: make test                            # execute `make test`...
      from: stable:aws-ebs-csi-driver-operator-test  # ...inside the specified image
      resources:             
        requests:
          cpu: 100m
    workflow: ipi-aws        # `ipi-$PLATFORM` workflows implement the "usual" OpenShift installation
```

#### Gotchas

These are the possible problems you may encounter when porting an
`openshift_installer_custom_test_image` test to a multi-stage workflow.

##### Tests rely on injected `openshift-tests` binary

The `openshift_installer_custom_test_image` template silently injected
`openshift-tests` binary to the specified image. This was a hack and will not
be implicitly supported in multi-stage workflows. 

**Resolution:** Tests that need `openshift-tests` binary should use a custom image
where that binary is present. This can be achieved by explicitly injecting that
binary into the desired image and using the resulting image to run the tests.

