# Windows Support

## Index

- [About](#about)
- [Consul k8s Control Plane Changes](#consul-k8s-control-plane-changes)
  - [Enabling Pod OS detection](#enabling-pod-os-detection)
  - [Assigning Values Depending on the OS](#assigning-values-depending-on-the-os)
- [Consul Helm Chart Changes](#consul-helm-chart-changes)
  - [Changes in values.yaml](#changes-in-valuesyaml)
  - [Changes in connect-injector-deployment.yaml Template](#changes-in-connect-injector-deploymentyaml-template)

## About

The purpose of this file is to document how we achieved automatic Consul injection into Windows workloads.

## Consul k8s Control Plane Changes

### Enabling Pod OS detection

To enable Consul injection into Windows workloads, the first thing we needed was to detect the pod's OS. Using Go's k8s.io/api/core/v1 package we can access the pod's **nodeSelector** field.
nodeSelector is the simplest recommended form of node selection constraint. You can add the nodeSelector field to your Pod specification and specify the node labels you want the target node to have. Kubernetes only schedules the Pod onto nodes that have each of the labels you specify (read more [here](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)).  
When running mixed clusters (Linux and Windows nodes), you must use the nodeSelector field to make sure the pods are deployed in the correct nodes.  
nodeSelector example:

```yml
nodeSelector:
  kubernetes.io/os: "windows"
```

This is why we chose to use nodeSelector to detect the pods' OS. When running Linux-only clusters, you may not use nodeSelector, in these cases using k8s.io/api/core/v1 `pod.Spec.NodeSelector["kubernetes.io/os"]` will return the field's zero value (an empty string ""). Knowing this, we created the **isWindows** function, the function simply returns true if the pod's OS is Windows and false if it isn't.

```go
func isWindows(pod corev1.Pod) bool {
  podOS := pod.Spec.NodeSelector["kubernetes.io/os"]
  return podOS == "windows"
}
```

### Assigning Values Depending on the OS

After successfully detecting the pod's OS, we used the function we created to assign values depending on the OS the pod was running, so the dataplane sidecar and init containers could be injected with valid values. These values are the result of previous work we had done on the subject (read more [here](https://github.com/hashicorp-education/learn-consul-k8s-windows/blob/main/WindowsTroubleshooting.md#encountered-issues)).  
Assigning values example:  

```go
var dataplaneImage, connectInjectDir string

if isWindows(pod) {
  dataplaneImage = w.ImageConsulDataplaneWindows
  connectInjectDir = "C:\\consul\\connect-inject"
} else {
  dataplaneImage = w.ImageConsulDataplane
  connectInjectDir = "/consul/connect-inject"
}
```

As you can see in the example above, we also updated the **MeshWebhook struct** in [mesh_webhook.go](./control-plane/connect-inject/webhook/mesh_webhook.go), adding the Windows images fields we require.

### Adding New Config Flags

To be able to set the values for **ImageConsulDataplaneWindows** and **ImageConsulK8SWindows** through the Helm chart, we created 3 new flags for the **inject-connect** subcommand.  
The new flags that enable setting this values are:

- consul-image-windows
- consul-dataplane-image-windows
- consul-k8s-image-windows

> **Warning**
> These flags require a default value, just like their Linux counterpart do, otherwise the `validateFlags()` function (line 750 in [command.go](./control-plane/subcommand/inject-connect/command.go)) will throw an error.  

## Consul Helm Chart Changes

### Changes in values.yaml

New fields were added as part of the **global** stanza:

- global.imageWindows: sets the default Windows Consul image.
- global.imageK8SWindows: sets the default Windows Consul K8s control-plane image.
- global.imageConsulDataplaneWindows: sets the default Windows Consul dataplane image.

### Changes in connect-injector-deployment.yaml Template

In order to pass the Windows image values set on the Helm chart to the connect-injector command flags, we modified the [connect-inject-deployment.yaml](./charts/consul/templates/connect-inject-deployment.yaml) template. We added the newly created config flags and used Helm's templating language to insert the required values.

> **Warning**
> It is essential for these changes to work, to have Windows images available for: Consul, consul-k8s-control-plane, consul-dataplane. You can read more about this [here](https://github.com/hashicorp-education/learn-consul-k8s-windows/tree/main/k8s-v1.0.x/dockerfiles).
