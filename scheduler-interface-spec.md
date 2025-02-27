<!--
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 -->

# Scheduler Interface Spec

Authors: The Yunikorn Scheduler Authors

## Objective

To define a standard interface that can be used by different types of resource management systems such as YARN/K8s.

### Goals for minimum viable product (MVP)

- Interface and implementation should be resource manager (RM) agnostic.
- Interface can handle multiple types of resource managers from multiple zones, and different policies can be configured in different zones.

Possible use cases:
- A large cluster needs multiple schedulers to achieve horizontally scalability.
- Multiple resource managers need to run on the same cluster. The managers grow and shrink according to runtime resource usage and policies.

### Non-Goals for minimum viable product (MVP)

- Handle process-specific information: Scheduler Interface only handles decisions for scheduling instead of how containers will be launched.

## Design considerations

Highlights:
- The scheduler should be as stateless as possible. It should try to eliminate any local persistent storage for scheduling decisions.
- When a RM starts, restarts or recovers the RM needs to sync its state with scheduler.

### Architecture

## Generic definitions

Interface and messages generic definition.

The syntax used for the declarations is `proto3`. The definition currently only provides go related info.
```protobuf
syntax = "proto3";
package si.v1;

import "google/protobuf/descriptor.proto";

option go_package = "lib/go/si";

extend google.protobuf.FieldOptions {
  // Indicates that a field MAY contain information that is sensitive
  // and MUST be treated as such (e.g. not logged).
  bool si_secret = 1059;
}
```

## Scheduler Interfaces

There are two kinds of interfaces, the first one is RPC based communication, the second one is API based.

RPC based, the [gRPC](https://grpc.io/) framework is used, will be useful when scheduler has to be deployed as a remote process.
For example when we need to deploy scheduler support multiple remote clusters.
A second example is when there is a cross language integration, like between Java and Go.

Unless specifically required we strongly recommend the use of the API based interface to avoid the overhead of the RPC serialization and de-serialization.

### RPC Interface

There are three sets of RPCs:

* **Scheduler Service**: RM can communicate with the Scheduler Service and do resource allocation/request update, etc.
* **Admin Service**: Admin can communicate with Scheduler Interface and get configuration updated.
* **Metrics Service**: Used to retrieve state of scheduler by users / RMs.

Currently only the design and implementation for the Scheduler Service is provided.

```protobuf
service Scheduler {
  // Register a RM, if it is a reconnect from previous RM the call will
  // trigger a cleanup of all in-memory data and resync with RM.
  rpc RegisterResourceManager (RegisterResourceManagerRequest)
    returns (RegisterResourceManagerResponse) { }

  // Update Scheduler status (this includes node status update, allocation request
  // updates, etc. And receive updates from scheduler for allocation changes,
  // any required status changes, etc.
  rpc Update (stream UpdateRequest)
    returns (stream UpdateResponse) { }
}

/*
service AdminService {
  // Include
  //   addQueueInfo.
  //   removeQueueInfo.
  //   updateQueueInfo.
  rpc UpdateConfig (UpdateConfigRequest)
    returns (UpdateConfigResponse) {}
}
*/

/*
service MetricsService {
}
*/
```
#### Why bi-directional gRPC

The reason of using bi-directional streaming gRPC is, according to performance benchmark: https://grpc.io/docs/guides/benchmarking.html latency is close to 0.5 ms.
The same performance benchmark shows streaming QPS can be 4x of non-streaming RPC.
Considering scheduler needs both throughput and better latency, we go with streaming API for scheduler related decisions.

### API Interface

The API interface only relies on the message definition and not on other generated code as the RPC Interface does.
Below is an example of the Scheduler Service as defined in the RPC. The SchedulerAPI is bi-directional and can be a-synchronous.
For the asynchronous cases the API requires a callback interface to be implemented in the resource manager.
The callback must be provided to the scheduler as part of the registration.


```golang
package api

import "github.com/apache/incubator-yunikorn-scheduler-interface/lib/go/si"

type SchedulerAPI interface {
    // Register a new RM, if it is a reconnect from previous RM, cleanup 
	// all in-memory data and resync with RM. 
	RegisterResourceManager(request *si.RegisterResourceManagerRequest, callback ResourceManagerCallback) (*si.RegisterResourceManagerResponse, error)

	// Update Scheduler status (including node status update, allocation request 
	// updates, etc. 
	Update(request *si.UpdateRequest) error

	// Notify scheduler to reload configuration and hot-refresh in-memory state based on configuration changes 
	ReloadConfiguration(clusterID string) error
}

// RM side needs to implement this API
type ResourceManagerCallback interface {
	RecvUpdateResponse(response *si.UpdateResponse) error
}
```

### Communications between RM and Scheduler

Lifecycle of RM-Scheduler communication

```
Status of RM in scheduler:

                            Connection     timeout
    +-------+      +-------+ loss +-------+      +---------+
    |init   |+---->|Running|+---->|Paused |+---->| Stopped |
    +-------+      +----+--+      +-------+      +---------+
         RM register    |                             ^
         with scheduler |                             |
                        +-----------------------------+
                                   RM voluntarilly
                                     Shutdown
```

#### RM register with scheduler

When a new RM starts, fails, it will register with scheduler. In some cases, scheduler can ask RM to re-register because of connection issues or other internal issues.

```protobuf
message RegisterResourceManagerRequest {
  // An ID which can uniquely identify a RM **cluster**. (For example, if a RM cluster has multiple manager instances for HA purpose, they should use the same information when do registration).
  // If RM register with the same ID, all previous scheduling state in memory will be cleaned up, and expect RM report full scheduling state after registration.
  string rmID = 1;

  // Version of RM scheduler interface client.
  string version = 2;

  // Policy group name:
  // This defines which policy to use. Policy should be statically configured. (Think about network security group concept of ec2).
  // Different RMs can refer to the same policyGroup if their static configuration is identical.
  string policyGroup = 3;
}

// Upon success, scheduler returns RegisterResourceManagerResponse to RM, otherwise RM receives exception.
message RegisterResourceManagerResponse {
  // Intentionally empty.
}
```

#### RM and scheduler updates.

Below is overview of how scheduler/RM keep connection and updates.

```protobuf
message UpdateRequest {
  // New allocation requests or replace existing allocation request (if allocationID is same)
  repeated AllocationAsk asks = 1;

  // Allocations can be released.
  AllocationReleasesRequest releases = 2;

  // New node can be scheduled. If a node is notified to be "unscheduable", it needs to be part of this field as well.
  repeated NewNodeInfo newSchedulableNodes = 3;

  // Update nodes for existing schedulable nodes.
  // May include:
  // - Node resource changes. (Like grows/shrinks node resource)
  // - Node attribute changes. (Including node-partition concept like YARN, and concept like "local images".
  //
  // Should not include:
  // - Allocation-related changes with the node.
  // - Realtime Utilizations.
  repeated UpdateNodeInfo updatedNodes = 4;

  // ID of RM, this will be used to identify which RM of the request comes from.
  string rmID = 5;

  // RM should explicitly add application when allocation request also explictly belongs to application.
  // This is optional if allocation request doesn't belong to a application. (Independent allocation)
  repeated AddApplicationRequest newApplications = 6;

  // RM can also remove applications, all allocation/allocation requests associated with the application will be removed
  repeated RemoveApplicationRequest removeApplications = 7;
}

message UpdateResponse {
  // Scheduler can send action to RM.
  enum ActionFromScheduler {
    // Nothing needs to do
    NOACTION = 0;

    // Something is wrong, RM needs to stop the RM, and re-register with scheduler.
    RESYNC = 1;
  }

  // What RM needs to do, scheduler can send control code to RM when something goes wrong.
  // Don't use/expand this field for other general purposed actions. (Like kill a remote container process).
  ActionFromScheduler action = 1;

  // New allocations
  repeated Allocation newAllocations = 2;

  // Released allocations, this could be either ack from scheduler when RM asks to terminate some allocations.
  // Or it could be decision made by scheduler (such as preemption or timeout).
  repeated AllocationRelease releasedAllocations = 3;

  // Released allocation asks(placeholder), when the placeholder allocation times out
  repeated AllocationAskRelease releasedAllocationAsks = 4;

  // Rejected allocation requests
  repeated RejectedAllocationAsk rejectedAllocations = 5;

  // Rejected Applications
  repeated RejectedApplication rejectedApplications = 6;

  // Accepted Applications
  repeated AcceptedApplication acceptedApplications = 7;

  // Updated Applications
  repeated UpdatedApplication updatedApplications = 8;

  // Rejected Node Registrations
  repeated RejectedNode rejectedNodes = 9;

  // Accepted Node Registrations
  repeated AcceptedNode acceptedNodes = 10;
}

message UpdatedApplication {
  // The application ID that was updated
  string applicationID = 1;
  // State of the application
  string state = 2;
  // Timestamp of the state transition
  int64 stateTransitionTimestamp = 3;
  // Detailed message
  string message = 4;
}

message RejectedApplication {
  // The application ID that was rejected
  string applicationID = 1;
  // A human-readable reason message
  string reason = 2;
}

message AcceptedApplication {
  // The application ID that was accepted
  string applicationID = 1;
}

message RejectedNode {
  // The node ID that was rejected
  string nodeID = 1;
  // A human-readable reason message
  string reason = 2;
}

message AcceptedNode {
  // The node ID that was accepted
  string nodeID = 1;
}
```

#### Ask for more resources

Lifecycle of AllocationAsk:

```
                           Rejected by Scheduler
             +-------------------------------------------+
             |                                           |
             |                                           v
     +-------+---+ Asked  +-----------+Scheduler or,+-----------+
     |Initial    +------->|Pending    |+----+----+->|Rejected   |
     +-----------+By RM   +-+---------+ Asked by RM +-----------+
                                +
                                |
                                v
                          +-----------+
                          |Allocated  |
                          +-----------+
```

Lifecycle of Allocations:

```
         +--Allocated by
         v    Scheduler
 +-----------+        +------------+
 |Allocated  |+------ |Completed   |
 +---+-------+ Stoppe +------------+
     |         by RM
     |                +------------+
     +--------------->|Preempted   |
     +  Preempted by  +------------+
     |    Scheduler
     |
     |
     |                +------------+
     +--------------->|Expired     |
         Timeout      +------------+
        (Part of Allocation
           ask)
```

Common fields for allocation:

```protobuf
message Priority {
  oneof priority {
    // Priority of each ask, higher is more important.
    // How to deal with Priority is handled by each scheduler implementation.
    int32 priorityValue = 1;

    // PriorityClass is used for app owners to set named priorities. This is a portable way for
    // app owners have a consistent way to setup priority across clusters
    string priorityClassName = 2;
  }
}

// A sparse map of resource to Quantity.
message Resource {
  map<string, Quantity> resources = 1;
}

// Quantity includes a single int64 value
message Quantity {
  int64 value = 1;
}
```

Allocation ask:

```protobuf
message AllocationAsk {
  // Allocation key is used by both of scheduler and RM to track allocations.
  // It doesn't have to be same as RM's internal allocation id (such as Pod name of K8s or ContainerID of YARN).
  // Allocations from the same AllocationAsk which are returned to the RM at the same time will have the same allocationKey.
  // The request is considered an update of the existing AllocationAsk if an ALlocationAsk with the same allocationKey 
  // already exists.
  string allocationKey = 1;
  // The application ID this allocation ask belongs to
  string applicationID = 2;
  // The partition the application belongs to
  string partitionName = 3;
  // The amount of resources per ask
  Resource resourceAsk = 4;
  // Maximum number of allocations
  int32 maxAllocations = 5;
  // Priority of ask
  Priority priority = 6;
  // Execution timeout: How long this allocation will be terminated (by scheduler)
  // once allocated by scheduler, 0 or negative value means never expire.
  int64 executionTimeoutMilliSeconds = 7;
  // A set of tags for this spscific AllocationAsk. Allocation level tags are used in placing this specific
  // ask on nodes in the cluster. These tags are used in the PlacementConstraints.
  // These tags are optional.
  map<string, string> tags = 8;
  // The name of the TaskGroup this ask belongs to
  string taskGroupName = 9;
  // Is this a placeholder ask (true) or a real ask (false), defaults to false
  // ignored if the taskGroupName is not set
  bool placeholder = 10;
}
```

Application requests:

```protobuf
message AddApplicationRequest {
  // The ID of the application, must be unique
  string applicationID = 1;
  // The queue this application is requesting. The scheduler will place the application into a
  // queue according to policy, taking into account the requested queue as per the policy.
  string queueName = 2;
  // The partition the application belongs to
  string partitionName = 3;
  // The user group information of the application owner
  UserGroupInformation ugi = 4;
  // A set of tags for the application. These tags provide application level generic inforamtion.
  // The tags are optional and are used in placing an appliction or scheduling.
  // Application tags are not considered when processing AllocationAsks.
  map<string, string> tags = 5;
  // Execution timeout: How long this application can be in a running state
  // 0 or negative value means never expire.
  int64 executionTimeoutMilliSeconds = 6;
  // The total amount of resources gang placeholders will request
  Resource placeholderAsk = 7;
  // Gang scheduling style can be hard (the application will fail after placeholder timeout)
  // or soft (after the timeout the application will be scheduled as a normal application)
  string gangSchedulingStyle = 8;
}

message RemoveApplicationRequest {
  // The ID of the application to remove
  string applicationID = 1;
  // The partition the application belongs to
  string partitionName = 2;
}
```

User information:
The user that owns the application. Group information can be empty. If the group information is empty the groups will be resolved by the scheduler when needed. 
```protobuf
message UserGroupInformation {
  // the user name
  string user = 1;
  // the list of groups of the user, can be empty
  repeated string groups = 2;
}
```

### Allocation of resources

The Allocation message is used in two cases:
1. A recovered allocation send from the RM to the scheduler
2. A newly created allocation from the scheduler.

```protobuf
message Allocation {
  // AllocationKey from AllocationAsk
  string allocationKey = 1;
  // Allocation tags from AllocationAsk
  map<string, string> allocationTags = 2;
  // UUID of the allocation
  string UUID = 3;
  // Resource for each allocation
  Resource resourcePerAlloc = 5;
  // Priority of ask
  Priority priority = 6;
  // Queue which the allocation belongs to
  string queueName = 7;
  // Node which the allocation belongs to
  string nodeID = 8;
  // The ID of the application
  string applicationID = 9;
  // Partition of the allocation
  string partitionName = 10;
  // The name of the TaskGroup this allocation belongs to
  string taskGroupName = 11;
  // Is this a placeholder allocation (true) or a real allocation (false), defaults to false
  // ignored if the taskGroupName is not set
  bool placeholder = 12;
}
```

#### Release previously allocated resources

```protobuf
message AllocationReleasesRequest {
  // The allocations to release
  repeated AllocationRelease allocationsToRelease = 1;
  // The asks to release
  repeated AllocationAskRelease allocationAsksToRelease = 2;
}

enum TerminationType {
    STOPPED_BY_RM = 0;          // Stopped or killed by ResourceManager (created by RM)
    TIMEOUT = 1;                // Timed out based on the executionTimeoutMilliSeconds (created by core)
    PREEMPTED_BY_SCHEDULER = 2; // Preempted allocation by scheduler (created by core)
    PLACEHOLDER_REPLACED = 3;   // Placeholder allocation replaced by real allocation (created by core)
}

// Release allocation: this is a bidirectional message. The Terminationtype defines the origin, or creator,
// as per the comment. The confirmation or response from the receiver is the same message with the same
// termination type set.  
message AllocationRelease {

  // The name of the partition the allocation belongs to
  string partitionName = 1;
  // The application the allocation belongs to
  string applicationID = 2;
  // The UUID of the allocation to release, if not set all allocations are released for
  // the applicationID
  string UUID = 3;
  // Termination type of the released allocation
  TerminationType terminationType = 4;
  // human-readable message
  string message = 5;
}

// Release ask
message AllocationAskRelease {
  // Which partition to release the ask from, required.
  string partitionName = 1;
  // optional, when this is set, filter allocation key by application id.
  // when application id is set and allocationKey is not set, release all allocations key under the application id.
  string applicationID = 2;
  // optional, when this is set, only release allocation ask by specified
  string allocationkey = 3;
  // Termination type of the released allocation ask
  TerminationType terminationType = 4;
  // For human-readable message
  string message = 5;
}
```

#### Schedulable nodes registration and updates

State transition of node:

```
   +-----------+          +--------+            +-------+
   |SCHEDULABLE|+-------->|DRAINING|+---------->|REMOVED|
   +-----------+          +--------+            +-------+
         ^       Asked by      +     Aasked by
         |      RM to DRAIN    |     RM to REMOVE
         |                     |
         +---------------------+
              Asked by RM to
              SCHEDULE again
```

See protocol below:

Registration of a new node with the scheduler. If the node exists then the request will be rejected.
```protobuf
message NewNodeInfo {
  // ID of node, must be unique
  string nodeID = 1;
  // node attributes
  map<string, string> attributes = 2;
  // Schedulable Resource
  Resource schedulableResource = 3;
  // Occupied Resource
  Resource occupiedResource = 4;
  // Allocated resources, this will be added when node registered to RM (recovery)
  repeated Allocation existingAllocations = 5;
}
```

Update of a registered node with the scheduler. If the node does not exist the update will fail.
```protobuf
message UpdateNodeInfo {
  // Action from RM
  enum ActionFromRM {
    // Update node resources, attributes.
    UPDATE = 0;

    // Do not allocate new allocations on the node.
    DRAIN_NODE = 1;

    // Decomission node, it will immediately stop allocations on the node and
    // remove the node from schedulable lists.
    DECOMISSION = 2;

    // From Draining state to SCHEDULABLE state.
    // If node is not in draining state, error will be thrown
    DRAIN_TO_SCHEDULABLE = 3;
  }

  // ID of node, the node must exist to be updated
  string nodeID = 1;
  // New attributes of node, which will replace previously reported attribute.
  map<string, string> attributes = 2;
  // new schedulable resource, scheduler may preempt allocations on the
  // node or schedule more allocations accordingly.
  Resource schedulableResource = 3;
  // when the scheduler is co-exist with some other schedulers, some node
  // resources might be occupied (allocated) by other schedulers.
  Resource occupiedResource = 4;
  // Action to perform by the scheduler
  ActionFromRM action = 5;
}
```

#### Feedback from Scheduler

Following is feedback from scheduler to RM:

When allocation ask rejected by scheduler, information will be shared by scheduler.

```protobuf
message RejectedAllocationAsk {
  string allocationKey = 1;
  // The ID of the application
  string applicationID = 2;
  // A human-readable reason message
  string reason = 3;
}
```

### Following are constant of spec

Scheduler Interface attributes start with the si prefix. Such constants are for example known attribute names for nodes and applications.

```constants
// Constants for node attributes
const (
	ARCH                = "si/arch"
	HostName            = "si/hostname"
	RackName            = "si/rackname"
	OS                  = "si/os"
	InstanceType        = "si/instance-type"
	FailureDomainZone   = "si/zone"
	FailureDomainRegion = "si/region"
	LocalImages         = "si/local-images"
	NodePartition       = "si/node-partition"
)

// Constants for allocation attributes
const (
	ApplicationID  = "si/application-id"
	ContainerImage = "si/container-image"
	ContainerPorts = "si/container-ports"
)
```

Allocation tags are key-value pairs, where the key should contain a domain, and optionally a group part.
These parts should precede the name of the key (and should be in that order) and separated by a "/" character.
Example allocation key: `kubernetes.io/meta/namespace`.

```constants
// Constants for allocation tags
const (
	// Domains
	DomainK8s      = "kubernetes.io/"
	DomainYuniKorn = "yunikorn.apache.org/"

	// Groups
	GroupMeta       = "meta/"
	GroupLabel      = "label/"
	GroupAnnotation = "annotation/"

	// Keys
	KeyPodName   = "podName"
	KeyNamespace = "namespace"
)
```

### Scheduler plugin

SchedulerPlugin is a way to extend scheduler capabilities. Scheduler shim can implement such plugin and register itself to
yunikorn-core, so plugged function can be invoked in the scheduler core.

```protobuf
message PredicatesArgs {
    // allocation key identifies a container, the predicates function is going to check
    // if this container is eligible to be placed ont to a node.
    string allocationKey = 1;
    // the node ID the container is assigned to.
    string nodeID = 2;
    // run the predicates for alloactions (true) or reservations (false)
    bool allocate = 3;
}

message ReSyncSchedulerCacheArgs {
   // a list of assumed allocations, this will be sync'd to scheduler cache.
   repeated AssumedAllocation assumedAllocations = 1;
   // a list of allocations to forget
   repeated ForgotAllocation forgetAllocations = 2;
}

message AssumedAllocation {
   // allocation key used to identify a container.
   string allocationKey = 1;
   // the node ID the container is assumed to be allocated to, this info is stored in scheduler cache.
   string nodeID = 2;
}

message ForgotAllocation {
   // allocation key used to identify a container.
   string allocationKey = 1;
}

message UpdateContainerSchedulingStateRequest {
   // container scheduling states
   enum SchedulingState {
     // the container is being skipped by the scheduler
     SKIPPED = 0;
     // the container is scheduled and it has been assigned to a node
     SCHEDULED = 1;
     // the container is reserved on some node, but not yet assigned
     RESERVED = 2;
     // scheduler has visited all candidate nodes for this container
     // but non of them could satisfy this container's requirement
     FAILED = 3;
   }

   // application ID
   string applicartionID = 1;
   
   // allocation key used to identify a container.
   string allocationKey = 2;

   // container scheduling state
   SchedulingState state = 3;

   // an optional plain message to explain why it is in such state
   string reason = 4;
}

message UpdateConfigurationRequest {
    // New config what needs to be saved
    string configs = 1;
}

message UpdateConfigurationResponse {
    // flag that marks the config update success or failure
    bool success = 1;
    
    // the old configuration what was changed
    string oldConfig = 2;
    
    // reason in case of failure
    string reason = 3;
}
```

#### Event Plugin

The Event Cache is a SchedulerPlugin that exposes events about scheduler objects aiming to help the end user to
see these events from the shim side. An event is sent to the shim through the callback as an `EventRecord`.
An `EventRecord` consists of the following fields:

```protobuf
message EventRecord {
   enum Type {
      REQUEST = 0;
      APP = 1;
      NODE = 2;
      QUEUE = 3;
   }

   // the type of the object associated with the event
   Type type = 1;
   // ID of the object associated with the event
   string objectID = 2;
   // the group this object belongs to
   // it specifies the application ID for allocations and the queue for applications
   string groupID = 3;
   // the reason of this event
   string reason = 4;
   // the detailed message as string
   string message = 5;
   // timestamp of the event
   int64 timestampNano = 6;
}
```

