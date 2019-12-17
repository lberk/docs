---
title: "Design and Theory Behind an Event Source"
linkTitle: "Design of an Event Source"
weight: 10
type: "docs"
---

# Topics
What are the personas and critical paths?
* Contributor: implement a new source with minimal k8s overhead (don't have to learn controller/k8s internals)
* Operator: easily install Sources and verify that they are "safe"
* Developer: easily discover what Sources they can pull from on this cluster
* Developer: easily configure a Source based on existing knowledge of other Sources.

## Separation of concerns
### Contributor:
* Receive Adapter(RA) - process that receives incoming events.
* Implement CloudEvent binding interfaces, we provide libraries for standard access to config.
* Configuration description (yaml, Go Struct, JSON?) links RA to controller runtime.

### Source library (provided by Knative):
* Controller runtime (this is what we share via injection) incorporates protocol specific config into "generic controller" CRD.
* Identifying event aspects to pass along to the serverless system
* Propagating events internally to the system (ie, cloudevents)

# Theory
Quick Introduction to Knative Eventing Sources
A Source is any Kubernetes object that generates or imports an event and relays that event to another endpoint on the cluster via [CloudEvents](https://github.com/cloudevents/spec/blob/v1.0/primer.md).

[The specification](https://github.com/knative/eventing/blob/master/docs/spec/sources.md)
for Knative Eventing Sources contains a number of requirements that
together define a well-behaved Knative Source


To achieve this, there are several separations of concerns that we have to keep in mind:
1. A controller to run our Event Source and reconcile the underlying deployments
2. A ‘receive adapter’ which imports the actual events
3. A series of identifying characteristics for our event
4. Transporting a valid event to the serverless system for further processing

There are also two different classes of developer to consider:
1. A "contributor" knows about the foreign protocol but not a Knative expert.
2. Knative-eventing expert knows how knative eventing components are implemented, configured and deployed, but is not an expert in all the foreign protocols that sources may implement.
These two roles will often not be the same person.  We want to confine the job of the "contributor" to implementing the `Receive Adapter`, and specifying what configuration their adapter needs connect, subscribe, or do whatever it does.

The Knative-eventing developer exposes protocol configuration as part of the `CRD`, and the controller passes configuration (which may include resolved data like URLs) to the `Recieve Adapter`.

API Resources required
* `KubeClientSet.Appsv1.Deployment` (Inherited via eventing base reconciler)
Used to deploy the Receive Adapter for ‘importing’ events
* `EventingClientSet.EventingV1Alpha1` (Inherited via eventing base reconciler)
Used to interact with Events within the knative system
* `SourceClientSet.SourcesV1Alpha1`
Used for source -- in this case, `samplesource` -- specific config and translated to the underlying deployment (via the inherited KubeClientSet)

To ease writing a new event source, the eventing subsystem has offloaded several core functionalities (via injection) to the `eventing-sources-controller`.


Fig 1. - Via shared [Knative Dependency Injection](https://docs.google.com/presentation/d/1aK5xCBv7wbfdDZAvnUE4vGWGk77EYZ6AbL0OR1vKPq8/edit#slide=id.g596dcbbefb_0_40)
presentation in the knative community drive


Specifically, the `clientset`, `cache`, `informers`, and `listers` can all be generated and shared. Thus, they can be generated, imported, and assigned to the underlying reconciler when creating a new controller source implementation:

```golang
import (
…
sampleSourceClient "knative.dev/sample-source/pkg/client/injection/client"
samplesourceinformer “knative.dev/sample-source/pkg/client/injection/informers/samples/v1alpha1/samplesource"
)
…
sampleSourceInformer := samplesourceinformer.Get(ctx)

r := &Reconciler{
...
		samplesourceClientSet: sampleSourceClient.Get(ctx),
		samplesourceLister:    sampleSourceInformer.Lister(),
...
```
Ensure that the specific source subdirectory has been added to the injection portion of the `hack/update-codegen.sh` script.

File Layout & Hierarchy
* `cmd/controller/main.go` - Pass source’s NewController implementation to the shared main
* `cmd/recieve_adapter/main.go` - Translate resource variables to underlying adapter struct (to eventually be passed into the serverless system) 
* `pkg/reconciler/controller.go` - NewController implementation to pass to sharedmain
* `pkg/reconciler/samplesource.go` - reconciliation functions for the receive adapter
* `pkg/apis/samples/VERSION/samplesource_types.go` - schema for the underlying api types (variables to be defined in the resource yaml)
* `pkg/apis/samples/VERSION/samplesource_lifecycle.go` - status updates for the source’s reconciliation details
  * Source ready
  * Sink provided
  * Deployed
  * Eventtype Provided
  * K8s Resources Correct
* `pkg/adapter/adapter.go` - receive_adapter functions supporting translation of events to CloudEvents