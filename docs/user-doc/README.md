# Setup and Configure WASM Extension

The following page explains how to setup and configure the WASM extension for the Kubernetes API-Server.

First you have to setup a Kubernetes cluster which runs with the custom build of the Kubernetes API-Server which contains the WASM extension.
See [cluster setup documentation](../cluster-setup/) on how to setup and run a Kubernetes cluster with a custom build of an API-Server.

If you don't want to build the API-Server by your own you can use the following image:
* `dvob/kube-apiserver:wasm` (`dvob/kube-apiserver@sha256:69f9bc68e50bffb0db5ed105ee10b8098adc5a029449ad91543eb97e37440f15`)

To configure the WASM extension you have to prepare configuration files and the actual WASM modules.
Copy the files to the server which runs your API-Server.
In a typical kubeadm setup you also have to update the kube-apiserver mainfest to mount the files from your server into the API-Server Pod.

The easiest way to mount all required files into the apiserver Pod is to place all files in one directory and mount that directory into the API-Server.
For this you have to extend `/etc/kubernetes/manifests/kube-apiserver.yaml` with the following parts:
```yaml
# spec.containers[0].volumeMounts
    - mountPath: /etc/kubernetes/wasm
      name: wasm
      readOnly: true

# spec.volumes
  - hostPath:
      path: /etc/kubernetes/wasm
      type: DirectoryOrCreate
    name: wasm
```

For all use cases (Authentication, Authorization and Admission) you configure a list of WASM modules which should be consulted:
```yaml
modules:
- module: /etc/kubernetes/wasm/my_module1.wasm
- module: /etc/kubernetes/wasm/my_module2.wasm
```

All use cases use the same module basic configuration.
```yaml
# modules specifies the modules which should be consulted to perform the action.
modules:
- # name (optional) specifies the name to identify the wasm module (e.g. in log messages).
  # if not specified it defaults to the basename of the module path (in this example 'file.wasm')
  name: my-module-1

  # module is the path to the wasm file
  module: /path/to/the/file.wasm

  # settings (optional) for the module. these are passed with each invocation
  # (see module specification).
  settings: {}

  # debug (optional) enables debug output. this prints all inputs and output
  # which are passed between the host and the module.
  debug: false
```

The admission modules support additional configurations (see below).

If you change the configuration you have to restart the API-Server to apply the changes.

# Authentication
To enable the WASM authentication you have to configure the following option on the API-Server:
```
--authentication-wasm-config-file=/etc/kubernetes/wasm/authn.conf
```

The authentication extension consults each module in the module list until one successfully authenticates the token.

## Authentication Example
`/etc/kubernetes/wasm/authn.conf`:
```yaml
modules:
- name: magic-authentication
  module: /etc/kubernetes/wasm/magic_authenticator.wasm
```

Copy the module file from https://github.com/dvob/k8s-wasi-rs/releases/download/v0.1.1/magic_authenticator.wasm to `/etc/kubernetes/wasm/magic_authenticator.wasm`.

# Authorization
To enable the WASM authorization you have to add `WASM` to the authorization modes and specify a modules configuration:
```
--authorization-mode=Node,RBAC,WASM
--authorization-wasm-config-file=/etc/kubernetes/wasm/authz.conf
```

The authorization extension consults each module in the module list until one successfully authorizes the request.

## Authorization Example
`/etc/kubernetes/wasm/authz.conf`:
```yaml
modules:
- name: magic-authorization
  module: /etc/kubernetes/wasm/magic_authorizer.wasm
```

Copy the module file from https://github.com/dvob/k8s-wasi-rs/releases/download/v0.1.1/magic_authorizer.wasm to `/etc/kubernetes/wasm/magic_authenticator.wasm`.


# Admission
To enable the WASM admission you have to add the `WASM` admission controller to the list of enabled admission plugins `--enable-admission-plugins`.
To configure the WASM admission controller you have to provide the configuration with the admission control config file `--admission-control-config-file`.
```
--enable-admission-plugins=WASM
--admission-control-config-file=/etc/kubernetes/wasm/admission.conf
```

The admission configuration supports the same basic module configuration as described above. The following additional settings are supported for admission modules:
```yaml
modules:
- module: /path/to/module.wasm

  # mutating (optional) specifies if a module is mutating or not. If mutating is
  # set to true the module gets called during the mutating phase of the admission.
  # By default mutating is false.
  mutating: false

  # type (optional) specifies the module type. Valid types are 'wasi' and 'kubewarden'.
  type: wasi

  # rules is a list of rules which define for which resources the admission
  # module should be called. See https://pkg.go.dev/k8s.io/api/admissionregistration/v1#RuleWithOperations
  # for documentation or look at the examples.
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["configmaps"]
```

If you specify the type `wasi` the module has to conform to the module specification.
If `kubewarden` is used as type the call logic described [here](https://docs.kubewarden.io/writing-policies/spec/intro-spec) is used to run the module.
You can find Kubewarden modules here: https://hub.kubewarden.io/

The WASM admission configuration is part of the full admission configuration and is either included as separate file or directly in the admission configuration.

File:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: WASM
  path: /path/to/admission-module-configuration.conf
```

Direct:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: WASM
  configuration:
    modules:
    - name: magic-validation
      module: /etc/kubernetes/wasm/magic_validator.wasm
      rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["configmaps"]
    - name: magic-mutator
      module: /etc/kubernetes/wasm/magic_mutator.wasm
      mutating: true
      rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["configmaps"]
```

## Admission Examples

### Example with Magic-Modules
`/etc/kubernetes/wasm/admission.conf`:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: WASM
  configuration:
    modules:
    - name: magic-validation
      module: /etc/kubernetes/wasm/magic_validator.wasm
      rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["configmaps"]
    - name: magic-mutator
      module: /etc/kubernetes/wasm/magic_mutator.wasm
      mutating: true
      rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["configmaps"]
```

Copy the module files the following module files to `/etc/kubernetes/wasm/`
* https://github.com/dvob/k8s-wasi-rs/releases/download/v0.1.1/magic_validator.wasm -> `/etc/kubernetes/wasm/magic_validator.wasm`
* https://github.com/dvob/k8s-wasi-rs/releases/download/v0.1.1/magic_mutator.wasm -> `/etc/kubernetes/wasm/magic_mutator.wasm`

### Example with Kubewarden Policy
In the following example we ensure that `configmaps` and `namespaces` have an annotation `puzzle.ch/owner`.
For this we use the Kubewarden policy safe-annotations: https://github.com/kubewarden/safe-annotations-policy.

`/etc/kubernetes/wasm/admission.conf`:
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: WASM
  configuration:
    modules:
    - name: safe-annotations
      type: kubewarden
      module: /etc/kubernetes/wasm/safe-annotations.wasm
      settings:
        mandatory_annotations:
        - puzzle.ch/owner
      rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources:
        - configmaps
        - namespaces
```

Copy https://github.com/kubewarden/safe-annotations-policy/releases/download/v0.2.0/policy.wasm to `/etc/kubernetes/wasm/safe-annotations.wasm`