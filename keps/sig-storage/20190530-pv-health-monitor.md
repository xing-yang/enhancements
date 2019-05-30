---
title: PV Health Monitor
authors:
  - "@NickrenREN"
owning-sig: sig-storage
participating-sigs:
  - sig-aaa
  - sig-bbb
reviewers:
  - "@msau42"
  - "@xing-yang"
  - "@jingxu97"
approvers:
  - "@msau42"
  - "@xing-yang"
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

For example: if volumes are unhealthy, and pods still try to write data to them, which will lead to data loss. 

And also if nodes break down, local PVs in the nodes therefore can not be accessed any more. 

So it is necessary to have a mechanism for monitoring PVs.

## User Experience

### User Cases

- If the local PV directory is deleted, users should know that and the local PV should be tainted;

- If the local PV path is not a mountpoint any more, the local PV should be tainted;

- If the node with local PVs in it breaks down, the local PVs should be tainted;

- If the volume provisioned by the driver is deleted, the PV object in kubernetes should be tainted;

- If we can not get access to the PV volume for a certain time (network or some other problems), we need to taint the PV;

- PV fsType checking ? bad blocks checking ?

- If an attached volume is no longer attached, the PV object should be tainted

- If a mounted volume is not longer mounted, the PV object should be tainted.


## Proposal

In this proposal, we only focus on PV monitoring. 

API change, local PVs and network PVs monitor are the three main parts we want to discuss here.

Regarding the API part, we don’t plan to change PV API object at the first stage, instead we may use the VolumeTaint introduced in this proposal . We propose to create three taints for PV monitoring, details can be found in Implementation section below.

For local PVs, we need to create an **agent** deployed to each node in the cluster to check the local PVs health condition and taint them if necessary.  A local PV monitor **controller** is also needed responsible for watching node failure events and tainting local PVs if nodes break down. BTW: we haven’t support local volumes in CSI, so CSI change is not needed here. 

For networked storage, we just need to create a **controller** to check the volumes’ status periodically, and each volume plugin should implement its own health checking methods. 
Since we are migrating in-tree volume plugins out of tree, the health checking methods should be supported in CSI spec. Detailed CSI spec info is [here](https://docs.google.com/document/d/1mjYlgXflAayLMPVFNT-qf3iJTVJYJBD-2RQh1jZTJ6s/edit#heading=h.3uxdz6kyha0s).

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

### Networked volume plugins

For networked storage,we can create a new controller called MonitorController just like the existing ProvisionController, which is responsible for calling each plugin’s CSI monitoring function periodically. 

And the monitoring method should be per volume plugin.

The detailed CSI changes will be found [here](https://docs.google.com/document/d/1mjYlgXflAayLMPVFNT-qf3iJTVJYJBD-2RQh1jZTJ6s/edit#heading=h.3uxdz6kyha0s) as listed in Proposal section.

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

