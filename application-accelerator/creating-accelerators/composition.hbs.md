
# Composition
## <a id="composition-intro"></a> Introduction

Despite their benefits, writing and maintaining accelerators can become repetitive and
verbose as new accelerators are added: some create a project different from the next but
with similar aspects, which requires some form of copy-paste.

To alleviate this concern, Application Accelerators support a feature named _Composition_
that allows re-use of parts of an accelerator, named **Fragments**.


## <a id="introducting-fragments"></a> Introducing Fragments

A **Fragment** looks exactly the same as an accelerator:

- it is made of a set of files
- contains an `accelerator.yaml` descriptor, with options declarations and a root Transform.

The only differences reside in the way they are declared to the system — they are
filed as **Fragments** custom resources — and the way they deal with files: because
they deal with their own files and files from the accelerator using them,
they typically use dedicated conflict resolution [strategies](transforms/conflict-resolution.md)
(more on that later).

Fragments may be thought of as "functions" in programming languages: once defined and
referenced, they may be "called" at various points in the main accelerator.
The composition feature is designed with ease-of-use and "common use case first"
in mind, so these "functions" are typically called with as little noise as possible,
you can also call them with complex or different values.

Composition relies on two building blocks that play hand in hand:

- the `imports` section at the top of an accelerator manifest,
- and the, `InvokeFragment` Transform, to be used alongside any other Transform.

## <a id="imports-section-explained"></a>| The `imports` section explained

To be usable in composition, a fragment MUST be _imported_ in the dedicated
section of an accelerator manifest:
```yaml
accelerator:
  name: my-awesome-accelerator
  options:
    - name: flavor
      dataType: string
      defaultValue: Strawberry
  imports:
    - name: my-first-fragment
    - name: another-fragment
engine:
  ...
```

The effect of importing a fragment this way is twofold:

- it makes its files available to the engine (this is why importing a fragment is required),
- it exposes all of its options as options of the accelerator, as if they were defined
  by the accelerator itself.

So in the above, example, if the `my-first-fragment` fragment had the following `accelerator.yaml`
file
```yaml
accelerator
  name: my-first-fragment
  options:
    - name: optionFromFragment
      dataType: boolean
      description: this option comes from the fragment

...
```

then it is as if the `my-awesome-accelerator` accelerator defined it:
```yaml
accelerator:
  name: my-awesome-accelerator
  options:
    - name: flavor
      dataType: string
      defaultValue: Strawberry
    - name: optionFromFragment
      dataType: boolean
      description: this option comes from the fragment
  imports:
    - name: my-first-fragment
    - name: another-fragment
engine:
  ...
```

All of the metadata about options (type, default value, description, choices if applicable, _etc._)
is coming along when being imported.

As a consequence of this, users are prompted for a value for those options that come
from fragments, as if they were options of the accelerator.

## <a id="using-invokefragment-transform"></a> Using the `InvokeFragment` Transform

The second part at play in composition is the `InvokeFragment` Transform.

As with any other transform, it may be used anywhere in the `engine` tree and
receives files that are "visible" at that point. Those files, alongside those
that make up the fragment are made available to the fragment logic. If the fragment
wants to choose between two versions of a file for a path, two new
conflict resolution [strategies](transforms/conflict-resolution.md) are available: `FavorForeign` and `FavorOwn`.

The behavior of the `InvokeFragment` Transform is very straight forward: after having validated
options that the fragment expects (and maybe after having set default values for
options that support one), it executes the whole Transform of the fragment _as if
it was written in place of `InvokeFragment`_.

See the `InvokeFragment` [reference page](transforms/invoke-fragment.md) for more explanations, examples, and configuration options. This topic now focuses on additional features of the `imports` section that are seldom used but still
available to cover more complex use-cases.

## <a id="back-to-imports"></a> Back to the `imports` section

The complete definition of the `imports` section is as follows, with
features in increasing order of "complexity":
```yaml
accelerator:
  name: ...
  options:
    - name: ...
    ...
  imports:
    - name: some-fragment

    - name: another-fragment
      expose:
        - name: "*"

    - name: yet-another-fragment
      expose:
        - name: someOption

        - name: someOtherOption
          as: aDifferentName
engine:
  ...
```
As shown earlier, the `imports` section calls a list of fragments to import and by default
all of their options become options of the accelerator. Those options appear _after_
the options defined by the accelerator, in the order the fragments are imported in.

**Note:** It is even possible for a fragment to import another fragment, the semantics
being the same as when an accelerator imports a fragment. This is a way to
break apart a fragment even further if needed.

When importing a fragment, you may select which options of the fragment
to make available as options of the accelerator. **This feature should only be used
when a name clash arises in option names.**

The semantics of the `expose` block are as follows:

- for every `name`/`as` pair, don't use the original (`name`) of the
  option but instead use the alias (`as`). Other metadata about the option
  is left unchanged.
- if the special `name: "*"` (which is NOT a legit option name usually) appears,
 all imported option names that are not remapped (the index at which the
  `*` appears in the yaml list is irrelevant) may be exposed
  with their original name.
- The default value for `expose` is `[{name: "*"}]`, _i.e._ by default
  expose all options with their original name.
- As soon as a single remap rule appears, the default is overridden (_i.e._
  to override some names AND expose the others unchanged, the `*` must
  be explicitly re-added)

### <a id="using-dependsOn-in-imports"></a> Using `dependsOn` in the `imports` section

Lastly, as a convenience for conditional use of fragments, you can make an exposed imported option _depend on_ another option, as in the following example:

```yaml
  imports:
    - name: tap-initialize
      expose:
        - name: gitRepository
          as: gitRepository
          dependsOn:
            name: deploymentType
            value: workload
        - name: gitBranch
          as: gitBranch
          dependsOn:
            name: deploymentType
            value: workload
```

This plays well with the use of `condition`, as in the following expample:

```yaml
...
engine:
  ...
    type: InvokeFragment
    condition: "#deploymentType == 'workload'"
    reference: tap-initialize```
```

## <a id="cli-fragments"></a> Discovering fragments using Tanzu CLI accelerator plug-in

Using the accelerator plug-in for Tanzu CLI, it is possible to list fragments that are available. Running the following command:

```sh
tanzu accelerator fragment list
```

should result in a list of available accelerator fragments:

```sh
NAME                                 READY   REPOSITORY
app-sso-client                       true    source-image: dev.registry.tanzu.vmware.com/app-accelerator/fragments/app-sso-client@sha256:ed5cf5544477d52d4c7baf3a76f71a112987856e77558697112e46947ada9241
java-version                         true    source-image: dev.registry.tanzu.vmware.com/app-accelerator/fragments/java-version@sha256:df99a5ace9513dc8d083fb5547e2a24770dfb08ec111b6591e98497a329b969d
live-update                          true    source-image: dev.registry.tanzu.vmware.com/app-accelerator/fragments/live-update@sha256:c2eda015b0f811b0eeaa587b6f2c5410ac87d40701906a357cca0decb3f226a4
spring-boot-app-sso-auth-code-flow   true    source-image: dev.registry.tanzu.vmware.com/app-accelerator/fragments/spring-boot-app-sso-auth-code-flow@sha256:78604c96dd52697ea0397d3933b42f5f5c3659cbcdc0616ff2f57be558650499
tap-initialize                       true    source-image: dev.registry.tanzu.vmware.com/app-accelerator/fragments/tap-initialize@sha256:7a3ae8f9277ef633200622dbf9d0f5a07dea25351ac3dbf803ea2fa759e3baac
tap-workload                         true    source-image: dev.registry.tanzu.vmware.com/app-accelerator/fragments/tap-workload@sha256:8056ad9f05388883327d9bbe457e6a0122dc452709d179f683eceb6d848338d0
```

The `tanzu accelerator fragment get <fragment-name>` command will show all the options defined for the fragment and also any accelerators or other fragments that import this fragment. Run this command:

```sh
tanzu accelerator fragment get java-version
```

and the following output should be displayed:

```sh
name: java-version
namespace: accelerator-system
displayName: Select Java Version
ready: true
options:
- choices:
  - text: Java 8
    value: "1.8"
  - text: Java 11
    value: "11"
  - text: Java 17
    value: "17"
  defaultValue: "11"
  inputType: select
  label: Java version to use
  name: javaVersion
  required: true
artifact:
  message: Resolved revision: dev.registry.tanzu.vmware.com/app-accelerator/fragments/java-version@sha256:df99a5ace9513dc8d083fb5547e2a24770dfb08ec111b6591e98497a329b969d
  ready: true
  url: http://source-controller-manager-artifact-service.source-system.svc.cluster.local./imagerepository/accelerator-system/java-version-frag-97nwp/df99a5ace9513dc8d083fb5547e2a24770dfb08ec111b6591e98497a329b969d.tar.gz
imports:
  None
importedBy:
  accelerator/java-rest-service
  accelerator/java-server-side-ui
  accelerator/spring-cloud-serverless
```

This shows the `options` as well as `importedBy` with a list of three accelerators that import this fragment.

Correspondingly, the `tanzu accelerator get <accelerator-name>` show the fragments that an accelerator imports. Run this command:

```sh
tanzu accelerator get java-rest-service
```

and the following ouput should be shown:

```sh
name: java-rest-service
namespace: accelerator-system
description: A Spring Boot Restful web application including OpenAPI v3 document generation and database persistence, based on a three-layer architecture.
displayName: Tanzu Java Restful Web App
iconUrl: data:image/png;base64,...abbreviated...
source:
  image: dev.registry.tanzu.vmware.com/app-accelerator/samples/java-rest-service@sha256:c098bb38b50d8bbead0a1b1e9be5118c4fdce3e260758533c38487b39ae0922d
  secret-ref: [{reg-creds}]
tags:
- java
- spring
- web
- jpa
- postgresql
- tanzu
ready: true
options:
- defaultValue: customer-profile
  inputType: text
  label: Module artifact name
  name: artifactId
  required: true
- defaultValue: com.example
  inputType: text
  label: Module group name
  name: groupId
  required: true
- defaultValue: com.example.customerprofile
  inputType: text
  label: Module root package
  name: packageName
  required: true
- defaultValue: customer-profile-database
  inputType: text
  label: Database Instance Name this Application will use (can be existing one in
    the cluster)
  name: databaseName
  required: true
- choices:
  - text: Maven (https://maven.apache.org/)
    value: maven
  - text: Gradle (https://gradle.org/)
    value: gradle
  defaultValue: maven
  inputType: select
  name: buildTool
  required: true
- choices:
  - text: Flyway (https://flywaydb.org/)
    value: flyway
  - text: Liquibase (https://docs.liquibase.com/)
    value: liquibase
  defaultValue: flyway
  inputType: select
  name: databaseMigrationTool
  required: true
- dataType: boolean
  defaultValue: false
  label: Expose OpenAPI endpoint?
  name: exposeOpenAPIEndpoint
- defaultValue: ""
  dependsOn:
    name: exposeOpenAPIEndpoint
  inputType: text
  label: System API Belongs To
  name: apiSystem
- defaultValue: ""
  dependsOn:
    name: exposeOpenAPIEndpoint
  inputType: text
  label: Owner of API
  name: apiOwner
- defaultValue: ""
  dependsOn:
    name: exposeOpenAPIEndpoint
  inputType: text
  label: API Description
  name: apiDescription
- choices:
  - text: Java 8
    value: "1.8"
  - text: Java 11
    value: "11"
  - text: Java 17
    value: "17"
  defaultValue: "11"
  inputType: select
  label: Java version to use
  name: javaVersion
  required: true
- dataType: boolean
  defaultValue: true
  dependsOn:
    name: buildTool
    value: maven
  inputType: checkbox
  label: Include TAP IDE Support for Java Workloads
  name: liveUpdateIDESupport
- defaultValue: dev.local
  dependsOn:
    name: liveUpdateIDESupport
  description: The prefix for the source image repository where source can be stored
    during development
  inputType: text
  label: The source image repository prefix to use when pushing the source
  name: sourceRepositoryPrefix
artifact:
  message: Resolved revision: dev.registry.tanzu.vmware.com/app-accelerator/samples/java-rest-service@sha256:c098bb38b50d8bbead0a1b1e9be5118c4fdce3e260758533c38487b39ae0922d
  ready: true
  url: http://source-controller-manager-artifact-service.source-system.svc.cluster.local./imagerepository/accelerator-system/java-rest-service-acc-wcn8x/c098bb38b50d8bbead0a1b1e9be5118c4fdce3e260758533c38487b39ae0922d.tar.gz
imports:
  java-version
  live-update
  tap-workload
```

Note the `imports` section at the end that shows the fragments that this accelerator imports. The `options` section shows all options that are defined for this accelerator. This includes all options that are defined in the imported fragments, e.g. the options for Java version that are imported from the `java-version` fragment.
