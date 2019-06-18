---
title: PV Health Monitor
authors:
  - "@NickrenREN"
  - "@xing-yang"
owning-sig: sig-storage
participating-sigs:
  - sig-aaa
  - sig-bbb
reviewers:
  - "@msau42"
  - "@jingxu97"
approvers:
  - "@msau42"
  - "@jingxu97"
  - "@saad-ali"
  - "@thockin"
editor: TBD
creation-date: 2019-05-30
last-updated: 2019-05-30
status: provisional

---

# PV Health Monitor

## Table of Contents

  * [Title](#title)
      * [Table of Contents](#table-of-contents)
      * [Motivation](#motivation)
      * [User Experience](#user-experience)
         * [Use Cases](#use-cases)
      * [Proposal](#proposal)
      * [Implementation](#implementation)
         * [API change](#api-change)
         * [Networked volume plugins](#networked-volume-plugins)
         * [Local storage](#local-storage)
         * [PV controller changes](#pv-controller-changes)
      * [Implementation History](#implementation-history)


## Motivation

For now, kubernetes has no way to monitor PVs, which may cause serious problems. 

For example: if volumes are unhealthy, and pods still try to write data to them, it can lead to data loss or signficantly reduced performance.

And also if nodes break down, local PVs in the nodes therefore can not be accessed any more. 

So it is necessary to have a mechanism for monitoring PVs.

## User Experience

### User Cases

#### Local PV

- If the local PV directory is deleted, users should know that and the local PV should be tainted;

- If the local PV path is not a mountpoint any more, the local PV should be tainted;

- If the node with local PVs in it breaks down, the local PVs should be tainted;

- If the volume provisioned by the driver is deleted, the PV object in kubernetes should be tainted;

- If we can not get access to the PV volume for a certain time (network or some other problems), we need to taint the PV;

- PV fsType checking? bad blocks checking?

#### Remote PV

- If the volume provisioned by the driver is deleted, the PV object in kubernetes should be tainted;

- If we can not get access to the PV volume for a certain time (network or some other problems), we need to taint the PV;

- If an attached volume is no longer attached, the PV object should be tainted

- If a mounted volume is not longer mounted, the PV object should be tainted.

- If volume is not usable, e.g., filesystem corruption, bad blocks, etc, the PV object should be tainted.

## Proposal

In this proposal, we only focus on PV monitoring. 

API change, local PVs and network PVs monitor are the three main parts we want to discuss here.

Regarding the API part, we don’t plan to change PV API object at the first stage, instead we may use the VolumeTaint introduced in [this](https://github.com/kubernetes/enhancements/pull/766) proposal . We propose to create three taints for PV monitoring, details can be found in Implementation section below.

For local PVs, we need to create an **agent** deployed to each node in the cluster to check the local PVs health condition and taint them if necessary.  A local PV monitor **controller** is also needed responsible for watching node failure events and tainting local PVs if nodes break down. BTW: we haven’t support local volumes in Container Storage Interface (CSI), so CSI change is not needed here.

For networked storage, we just need to create a **controller** to check the volumes’ status periodically, and each volume plugin should implement its own health checking methods. 
Since we are migrating in-tree volume plugins out of tree, the health checking methods should be supported in CSI.

The whole architecture is as below:
![pv health monitor architecture](./pv-health-monitor.png)


## Implementation

### API change

We plan to introduce three taints for PV health condition.

**One is `VolumeNotAttached` taint**,
```
Key: VolumeNotAttached 
Value: true
VolumeTaintEffect: NoEffect  (Make VolumeTaintEffect non-required field)
```

If the volume is not attached now, we need to taint the PV using the VolumeNotAttached taint. 

There is already a self-healing feature in-tree that checks if a volume is still attached.  It checks every minute.  If not attached, it tries to attach again. 

If the self-healing controller reattaches successfully, it should remove the VolumeNotAttached taint then. 

In order not to affect the self healing feature, we set the VolumeTaintEffect to NoEffect.

**Another one is `VolumeNotMounted` taint**,
```
Key: VolumeNotMounted 
Value: true
VolumeTaintEffect: NoEffect  (Make VolumeTaintEffect non-required field)
```

This can be used for both device mount and volume mount checking. If the volume is not mounted, we need to taint it.

For in use volumes, if the volumes are not mounted, reactions should be taken. 

The reactions needs further discussion, and is not in the scope of this doc. 

For unused volumes, the volume manager will try to mount again when we want to use it, so we do not need any effect.

In summary, we set the VolumeTaintEffect to NoEffect too.

**The last one is: `VolumeNotUsable` taint**,
```
Key: VolumeNotUsable (e.g. bad blocks, filesystem corrupted, etc)
Value: true
VolumeTaintEffect: NoAttachOrMount
```

If the volume is not usable because of bad blocks, filesystem corruption, disk deletion or any other reasons, we need to taint the PV using this taint.

Here, we use NoAttachOrMount effect mainly for multi-attach volumes. 

Because NoAttachOrMount effect will prevent new pods from using the tainted PV. 

And as mentioned above, for in use volumes, reactions are also needed.

When we finish discussing the reaction for unhealthy PVs, we need to revisit all the VolumeTaintEffects.

### Networked storage

For networked storage, the following volume health conditions will be checked:
* Checks if volume still exists and/or is attached
* Checks if volume is still mounted and usable

Container Storage Interface (CSI) specification will be modified to add two RPCs ControllerCheckVolume and NodeCheckVolume. A monitor controller (external-monitor) that is responsible for watching the PersistentVolumeClaim, PersistentVolume, and VolumeAttachment API objects will be added. The external-monitor can be deployed as a sidecar container together with the CSI driver.  For PVs, the external-monitor calls CSI driver’s monitor interface to check volume health status.

* ControllerCheckVolume RPC
  * It calls ControllerCheckVolume() to see if volume still exists; Taints PV with VolumeNotExisting if not existing any more.
  * If volume is attached, it calls ControllerCheckVolume() to see if volume is still attached; Taints PV with VolumeNotAttached if not attached any more.

* NodeCheckVolume RPC
  * For any PVC that is mounted, the external-monitor calls NodeCheckVolume() to see if volume is still mounted; Taints PV with VolumeNotMounted if not mounted any more.
  * Calls NodeCheckVolume to check if volume is usable, e.g., filesystem corruption, bad blocks, etc; Taint PV with VolumeNotUsable if volume is not usable.

CSI driver needs to implement the following CSI Controller RPC if CHECK_VOLUME controller capability is supported:
* ControllerCheckVolume()

CSI driver also needs to implement the following CSI Node RPC if CHECK_VOLUME node capability is supported:
* NodeCheckVolume()

The RPC changes needed in the CSI Spec are described in the following.

#### Add ControllerCheckVolume RPC

ControllerCheckVolume RPC checks if volume still exists and/or is still attached if attached already.

Input parameters volume_id and node_id are enough to detect whether a volume is still attached to the same node, but not enough to detect whether the attachment is compatible as before.

```
rpc ControllerCheckVolume (ControllerCheckVolumeRequest)
    returns (ControllerCheckVolumeResponse) {}
```

```
message ControllerCheckVolumeRequest {
  // The ID of the volume to be used on a node.
  // This field is REQUIRED.
  string volume_id = 1;

  // The ID of the node. This field is REQUIRED. The CO SHALL set this
  // field to match the node ID returned by `NodeGetInfo`.
  string node_id = 2;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the published volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a OPTIONAL field. (REQUIRED if checking whether attached or not)
  VolumeCapability volume_capability = 3; (may need this to check if compatible)

  // Indicates SP MUST publish the volume in readonly mode.
  // CO MUST set this field to false if SP does not have the
  // PUBLISH_READONLY controller capability.
  // This is a OPTIONAL field. (REQUIRED if checking whether attached or not)
  bool readonly = 4; (may need this to check if compatible)

  // Secrets required by plugin to complete controller publish volume
  // request. This field is OPTIONAL. Refer to the
  // `Secrets Requirements` section on how to use this field.
  map<string, string> secrets = 5 [(csi_secret) = true]; (Should not need secrets to check if it is attached, but need secrets to re-attach)

  // Volume context as returned by CO in CreateVolumeRequest. This field
  // is OPTIONAL and MUST match the volume_context of the volume
  // identified by `volume_id`.
  map<string, string> volume_context = 6; (may need this to check if compatible)
}
```

```
message ControllerCheckVolumeResponse {
  // Indicate whether the volume exists
  bool exists = 1;

  // Indicate whether the volume is attached
  bool is_attached = 2;
}
```

If volume is no longer attached, the monitoring (reaction) controller can trigger the CSI plugin to re-attach which may need secrets.  There are many reasons why the volume is no longer attached.  If it is caused by a network problem, re-attach may not work until the network problem is resolved.  The reaction controller will not be in the scope for the first phase of the volume health proposal.

If volume is still attached but not compatible, it needs admin intervention. Taint the volume, log an event but DO NOT attempt to detach and re-attach.

#### Add NodeCheckVolume RPC

NodeCheckVolume RPC checks if volume is still mounted and usable. To check whether a volume is usable, the CSI driver is supposed to check if filesystem is corrupted, whether there are bad blocks, etc. in this RPC.

```
rpc NodeCheckVolume (NodeCheckVolumeRequest)
    returns (NodeCheckVolumeResponse) {}
```

```
message NodeCheckVolumeRequest {
  // The ID of the volume to check. This field is REQUIRED.
  string volume_id = 1;

  // The CO SHALL set this field to the value returned by
  // `ControllerPublishVolume` if the corresponding Controller Plugin
  // has `PUBLISH_UNPUBLISH_VOLUME` controller capability, and SHALL be
  // left unset if the corresponding Controller Plugin does not have
  // this capability. This is an OPTIONAL field.
  map<string, string> publish_context = 2;

  // The path to which the volume was staged by `NodeStageVolume`.
  // It MUST be an absolute path in the root filesystem of the process
  // serving this request.
  // It MUST be set if the Node Plugin implements the
  // `STAGE_UNSTAGE_VOLUME` node capability.
  // This is an OPTIONAL field.
  string staging_target_path = 3;

  // The path to which the volume will be published. It MUST be an
  // absolute path in the root filesystem of the process serving this
  // request. The CO SHALL ensure uniqueness of target_path per volume.
  // The CO SHALL ensure that the parent directory of this path exists
  // and that the process serving the request has `read` and `write`
  // permissions to that parent directory.
  // For volumes with an access type of block, the SP SHALL place the
  // block device at target_path.
  // For volumes with an access type of mount, the SP SHALL place the
  // mounted directory at target_path.
  // Creation of target_path is the responsibility of the SP.
  // This is a REQUIRED field.
  string target_path = 4;

  // Volume capability describing how the CO intends to use this volume.
  // SP MUST ensure the CO can use the published volume as described.
  // Otherwise SP MUST return the appropriate gRPC error code.
  // This is a REQUIRED field.
  VolumeCapability volume_capability = 5;

  // Indicates SP MUST publish the volume in readonly mode.
  // This field is REQUIRED.
  bool readonly = 6;

  // Secrets required by plugin to complete node publish volume request.
  // This field is OPTIONAL. Refer to the `Secrets Requirements`
  // section on how to use this field.
  map<string, string> secrets = 7 [(csi_secret) = true];

  // Volume context as returned by CO in CreateVolumeRequest. This field
  // is OPTIONAL and MUST match the volume_context of the volume
  // identified by `volume_id`.
  map<string, string> volume_context = 8;
}

message NodeCheckVolumeResponse {
  // Indicate whether the volume is mounted
  bool is_mounted = 1;

  // Indicate whether the volume is usable
  bool usable = 2;
}
```

If volume is no longer mounted, monitoring (reaction) controller can trigger the CSI plugin to re-mount. The reaction controller will not be in the scope for the first phase of the volume health proposal.
If volume is still mounted but not compatible, it needs admin intervention. Log an event but DO NOT attempt to unmount and re-mount.

This RPC also checks whether volume is still usable by checking if filesystem is corrupted, if there are bad blocks, etc.


### Local storage

The local storage PV monitor should include two parts
 
- A daemonset on every node, which is responsible for monitoring local PVs in that specific node, no matter the PVs are created manually or by provisioner;

- A local PV monitor controller, which is responsible for watching node failure events. The controller can be deployed as deployment with leader election mechanism.

Detailed local PV monitoring method may be like:
```
 func (mon *minitor) CheckVolumeConditions(spec *v1.PersistentVolumeSpec) (string, error) {
            // check if PV is local storage
    if pv.Spec.Local == nil { 
		glog.Infof("PV: %s is not local storage", pv.Name)
		return ...
	}
            // check labels and selectors
               ...
            // check if PV belongs to this node
               ...
            // check if host dir still exists
               …
            // check if it is still mounted
                …
            // check if there are bad blocks...
}
```




### PV controller changes:

For unbound PVCs/PVs,  PVCs will not be bound to tainted PVs.



## Implementation History


- 20190530: KEP submitted

- Demo implementation (using annotations): 
https://github.com/NickrenREN/kubemonitor/tree/master/build/kube_storage_monitor/local_monitor
https://github.com/NickrenREN/kubemonitor/tree/master/build/kube_storage_monitor/node_watcher

