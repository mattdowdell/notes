# Ideas

zizmor-(something): GitHub Action to run zizmor in a docker container (pending container build by zizmor).

----

kubeswitch: CLI to switch namespace and context.

- Use bubbletea for choosing.
- Also allow values to be set via flags.

```sh
ksw [-c/--context CONTEXT] [-n/--namespace NAMESPACE]
```

- https://stackoverflow.com/questions/46926750/kubernetes-is-it-possible-to-configure-a-default-namespace-for-a-user (change namespace for context)
- `kubectl config use-context <name>` (change context)
