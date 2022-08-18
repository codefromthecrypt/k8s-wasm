# Kubernetes API Access Extensions with WebAssembly

This repository documents an extension of the Kubernetes API server which allows using WebAssembly modules to perform/extend the following actions:
* Authentication
* Authorization
* Admission (validating and mutating)

To implement this, I forked the Kubernetes source code and extended the API server accordingly: https://github.com/dvob/kubernetes/tree/wasm.

The extension is **not intended for production use**. It is a proof of concept to show how WebAssembly could be used to extend Kubernetes.

In the fork I created a new package [`pkg/wasm`](https://github.com/dvob/kubernetes/tree/wasm/pkg/wasm) in which I implemented an Authenticator, Authorizer and AdmissionController.
Most of the implementation lives in this new package.
There are only a few changes in the `pkg/kube-apiserver` package to add command line options enabling the WASM Authenticator, Authorizer and AdmissionController.

To run the WebAssembly modules we use the [Wazero](https://github.com/tetratelabs/wazero) runtime.
Wazero has zero dependencies and does not rely on CGO. Hence it can be easy integrated in a Go project without adding a ton of dependencies.

To pass data between our extension (host) and the WASM modules we make use of the capabilities of [WASI](https://wasi.dev/) (`fd_read`, `fd_write`).
The module reads the input data from standard input and writes the result to standard output.
See the [Module specification](./spec/) for the full details on how data is passed between host and modules.
For Admission the extension also supports to use [Kubewarden policies](https://hub.kubewarden.io/) which are not context aware.

See [Build and test Kubernetes API server](./docs/build-publish/) for a manual on how to build and test API server fork with the WASM extension.

See [User Documentation](./docs/user-doc/) for a full description on how to setup and configure the extended API server.

To implement your own modules see the [Module Specification](./spec/).
If you want to use Rust to implement the modules you can use the [k8s_wasi](https://github.com/dvob/k8s-wasi-rs) helper library.

## Links Overview
* [User Documentation](./docs/user-doc): How to setup and configure the WASM extension
* [Kubernetes Cluster Setup](./docs/cluster-setup/): How to setup a Kubernetes cluster
* [Module Specification](./spec/): 
  * [Rust module library](https://github.com/dvob/k8s-wasi-rs): Rust library which simplifies the creation of WASM modules according to the Module Specification.
* Experiments (PoCs)
  * [WASM](./wasm/)
    * [Runtimes](./wasm/runtime): Comparison of WebAssembly Runtimes
    * Data passing ([Modules](./wasm/modules/rs), [Runtimes](./wasm/runtime/))
  * [Kubernetes Integration](./k8s/)
    * [Webhook](./k8s/webhook/)
    * [Direct in API server](./k8s/api-server/)
