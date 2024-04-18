[![CI](https://github.com/aws/karpenter-provider-aws/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/aws/karpenter/actions/workflows/ci.yaml)
![GitHub stars](https://img.shields.io/github/stars/aws/karpenter-provider-aws)
![GitHub forks](https://img.shields.io/github/forks/aws/karpenter-provider-aws)
[![GitHub License](https://img.shields.io/badge/License-Apache%202.0-ff69b4.svg)](https://github.com/aws/karpenter-provider-aws/blob/main/LICENSE)
[![Go Report Card](https://goreportcard.com/badge/github.com/aws/karpenter-provider-aws)](https://goreportcard.com/report/github.com/aws/karpenter)
[![Coverage Status](https://coveralls.io/repos/github/aws/karpenter-provider-aws/badge.svg?branch=main)](https://coveralls.io/github/aws/karpenter?branch=main)
[![contributions welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat)](https://github.com/aws/karpenter-provider-aws/issues)

![](website/static/banner.png)

### This is a fork of karpenter-provider-aws. This shows the level of control we'd like to see for defining extended resources for the instance-types

In order to run this, you should clone both this repository and the forked core [karpenter](https://github.com/uniemimu/karpenter). Clone them to the same parent folder
so that the relative go.mod reference `../karpenter` is possible. In both of the repositories, checkout the same branch `crd` or `configmap`.

The gist of the changes are in the core repo, but you will see minor changes also here, like the `go.mod` replace statement.

To build, just do "make apply" and you should be good to go, assuming you had a cluster in AWS, with an already working karpenter setup which you are willing to upgrade.

The extension configurations are read top down in an array, and the first to match an instance name gets selected. Putting the more generic regexps matches to the bottom is the idea.

#### Configuration examples for the configmap -branches

Example 1 (configmap): if you want all instancetypes in all nodepools to have your special extended resource, you will achieve it like this:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: extension-config
  namespace: kube-system
data:
  instance_type_extensions.json: >
    [
      {
        "instanceNameRegex": ".*",
        "extendedResources": {
          "intel.com/special": 1
        }
      }
    ]
```

Example 2: (configmap): if you want a specific instancetype to have 2 of your special extended resources, but the rest of them to have just one,
regardless of what nodepool they are from:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: extension-config
  namespace: kube-system
data:
  instance_type_extensions.json: >
    [
      {
        "instanceNameRegex": "m6i\\.xlarge",
        "extendedResources": {
          "intel.com/special": 2
        }
      },
      {
        "instanceNameRegex": ".*",
        "extendedResources": {
          "intel.com/special": 1
        }
      }
    ]
```
Note the wonky escape for the dot. Regexp wants the dot to be escaped with `\` but json wants the `\` escaped with another `\`. Anyways, it matches `m6i.xlarge` and only that.

Example 3: (configmap): if you want instancetypes from your critical nodepool to have 2 of your special extended resources, but the rest of the nodepools to have just one:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: extension-config
  namespace: kube-system
data:
  instance_type_extensions.json: >
    [
      {
        "instanceNameRegex": ".*",
        "extendedResources": {
          "intel.com/special": 2
        },
        "requiredNodePoolLabels": {
            "intel.com/pool": "critical"
        }
      },
      {
        "instanceNameRegex": ".*",
        "extendedResources": {
          "intel.com/special": 1
        }
      }
    ]
```
Note that the nodepool of course needs to be configured to have the required label.

These examples tried to cover the possibility of defining simple configs for all instance types or to define individual instance type configurations for individual node pools,
or something in between.

Because regexps can be slow and there can be many instancetypes, the code in the core repo will only run the regexps once to find the extension for any instance type, and once it
has either found a match or figured out that none of the regexps match, it will use that stored result always. The example code does not have means of refreshing the config except via restarting of Karpenter, so there's still plenty room for improvements. This is just an example, not the solution.

#### Configuration examples for the crd -branches

The `crd` branches of the repositories allow for doing the same thing just via a custom resource definition. The benefit is having some validation for the configuration already
at the crd definition level, with the downside of being a larger change to the codebase.

Example 1 (crd): if you want all instancetypes in all nodepools to have your special extended resource, you will achieve it like this:
```yaml
apiVersion: karpenter.sh/v1beta1
kind: InstanceTypeExtension
metadata:
  name: test-extension
spec:
  extensions:
  - instanceNameRegex: ".*"
    extendedResources:
      intel.com/special: 1
```

Example 2: (crd): if you want a specific instancetype to have 2 of your special extended resources, but the rest of them to have just one,
regardless of what nodepool they are from:

```yaml
apiVersion: karpenter.sh/v1beta1
kind: InstanceTypeExtension
metadata:
  name: test-extension
spec:
  extensions:
  - instanceNameRegex: m6i\.xlarge
    extendedResources:
      intel.com/special: 2
  - instanceNameRegex: ".*"
    extendedResources:
      intel.com/special: 1
```
Note the escape for the dot. Regexp wants the dot to be escaped with `\`. Anyways, it matches `m6i.xlarge` and only that.

Example 3: (crd): if you want instancetypes from your critical nodepool to have 2 of your special extended resources, but the rest of the nodepools to have just one:
```yaml
apiVersion: karpenter.sh/v1beta1
kind: InstanceTypeExtension
metadata:
  name: test-extension
spec:
  extensions:
  - instanceNameRegex: ".*"
    extendedResources:
      intel.com/special: 2
    requiredNodePoolLabels:
      intel.com/pool: critical
  - instanceNameRegex: ".*"
    extendedResources:
      intel.com/special: 1
```
Note that the nodepool of course needs to be configured to have the required label.

### testing the extended resources

* Configure karpenter to have its instance-types extended by a new extended resource
* Define a pod which uses the new extended resource
* Deploy your pod
* Observe your pod is stuck pending
* Observe Karpenter creates a new node
* Add the new extended resource to the newly created node, e.g. use [curl](https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/#advertise-a-new-extended-resource-on-one-of-your-nodes)
* Observe your pod gets to running state
* Delete the pod
* Observe Karpenter deletes the now unnecessary node

___

Karpenter is an open-source node provisioning project built for Kubernetes.
Karpenter improves the efficiency and cost of running workloads on Kubernetes clusters by:

* **Watching** for pods that the Kubernetes scheduler has marked as unschedulable
* **Evaluating** scheduling constraints (resource requests, nodeselectors, affinities, tolerations, and topology spread constraints) requested by the pods
* **Provisioning** nodes that meet the requirements of the pods
* **Removing** the nodes when the nodes are no longer needed

Come discuss Karpenter in the [#karpenter](https://kubernetes.slack.com/archives/C02SFFZSA2K) channel, in the [Kubernetes slack](https://slack.k8s.io/) or join the [Karpenter working group](https://karpenter.sh/docs/contributing/working-group/) bi-weekly calls. If you want to contribute to the Karpenter project, please refer to the Karpenter docs.

Check out the [Docs](https://karpenter.sh/docs/) to learn more.

## Talks
- 09/08/2022 [Workload Consolidation with Karpenter](https://youtu.be/BnksdJ3oOEs)
- 05/19/2022 [Scaling K8s Nodes Without Breaking the Bank or Your Sanity](https://www.youtube.com/watch?v=UBb8wbfSc34)
- 03/25/2022 [Karpenter @ AWS Community Day 2022](https://youtu.be/sxDtmzbNHwE?t=3931)
- 12/20/2021 [How To Auto-Scale Kubernetes Clusters With Karpenter](https://youtu.be/C-2v7HT-uSA)
- 11/30/2021 [Karpenter vs Kubernetes Cluster Autoscaler](https://youtu.be/3QsVRHVdOnM)
- 11/19/2021 [Karpenter @ Container Day](https://youtu.be/qxWJRUF6JJc)
- 05/14/2021 [Groupless Autoscaling with Karpenter @ Kubecon](https://www.youtube.com/watch?v=43g8uPohTgc)
- 05/04/2021 [Karpenter @ Container Day](https://youtu.be/MZ-4HzOC_ac?t=7137)
