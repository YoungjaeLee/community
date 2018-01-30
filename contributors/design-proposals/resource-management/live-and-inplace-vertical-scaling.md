Live and In-place Vertical Scaling Proposal
================

_Authors:_
* @YoungjaeLee - Youngjae Lee &lt;leeyo@us.ibm.com&gt;
* @karthickrajamani - Karthick Rajamani &lt;karthick@us.ibm.com&gt;

# Abstract

This proposal is to enable the resizing of resources allocated to running containers of a pod without having to restart the pod.
The initial focus is on enabling this for Statefulsets, in particular, though this may be use-able for stand-alone/controller-free pods and Deployments as well.

# Motivation

Resources needed by containers can change over time for a variety of reasons - moving from live-test mode to production usage, change in user load or dataset sizes each of which again might come about for a variety of reasons.
Deployments and Statefulsets support the capability to change Request and Limit values specified for a container through supported pod spec update methods.
However, they currently require the pods be restarted to run with the new resource sizes.

There are a few reasons why restarting may not be desirable particularly for stateful services.

1. Moving or copying existing state information or re-sharding employing horizontal scaling may be expensive or even infeasible depending on the service or underlying application implementation.

2. Services with large amount of state associated with specific instances may encounter temporary loss in access to the specific state information while its corresponding container is being restarted. Or they might encounter non-trivial delays in getting back to expected performance.

3. State may be maintained on node-local storage/devices and restart might place the container/pod on a different node (unless suitably constrained). Might create more fragile scaling option for service and/or additional burden (to constrain placement) on user.

4. Where the restart is not needed, unnecessary load may be created on Kubernetes components that are responsible for pod deletion and creation potentially impacting Kubernetes scalability.
Resizing running containers of a pod in place could avoid that additional load.

Conversely, resizing without restart may not be the right option for all applications and the right option may also be resource dependent.
For example, an application might (or have the means to) assess resources available to it only at its initialization time and be unable to adjust its usage if those are dynamically changed.
Or it might be able to adjust to changes in certain resources (most can for CPU) but not for others (some application have particular difficulty freeing memory they have taken up).
So we need the means to express and implement the best approach for resizing for each workload and resource.

# Use Cases

* After deploying a service/application through `StatefulSet`, I want to change the amount of resources (for now, CPU and memory) allocated to each pod of the statefulset without restarting the pods.

# Objectives

1. Enable live and in-place resource resizing on a pod.
2. Add support for live and in-place resource resizing in `StatefulSet' controller.

# API and Usage

To express the policy for resizing in a pod spec, we introduce resource attribute `resizePolicy` with the following choices for value:
* RestartOnly. (the current behavior, default)
* LiveResizeable.

This attribute will be available per resource (such as cpu, memory) and so is adequate to indicate whether the workload can handle and prefer a change in each resource’s allocation for it without restarting.
With potentially multiple containers and multiple resizeable resources for each in a Pod, the response to an update of the pod spec will be determined by the a precedence order among the attribute values with RestartOnly dominating LiveResizeable, i.e., if two resources have been resized in the update to the spec and one of them has a policy of RestartOnly then the pod would be restarted to realize both updates.

We can optionally introduce an action annotation resourceResizeAction with following choices for value:
* Restart. (default)
* LiveResize.
* LiveResizePreferred.

If used, this would be included as part of the patch or appropriate update command providing the spec update for the resize.
It would indicate the preference of user at the time of resize.
Specifically, Restart for resourceResizeAction would indicate the pod be restarted for the corresponding resizing of resource(s), LiveResize would indicate the pod not be restarted the resize be realized live, and LiveResizePreferred would indicate that the resize be realized preferrably live but if that fails for any reason to accomplish it with a restart.

Usage of resizePolicy attribute in a pod spec:

```yaml
Resources:

