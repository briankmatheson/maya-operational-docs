# Replace a Failed cStor Disk

## 0. Creating a cStor Pool
* Show the disks.
`kubectl get bd -n openebs --show-labels`

* Create a claim.
```
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-disk-pool
spec:
  name: cstor-disk-pool
  type: disk
  poolSpec:
    poolType: mirrored
  blockDevices:
    blockDeviceList:
    - blockdevice-03d66e01c1fea3b985a55ed74bfd54be 
    - blockdevice-3b867be0018fc6c34b1c882ae1f6fa56
    - blockdevice-04698fb674505e050359184e4286f3b9
    - blockdevice-e43146de9b672e5814dbf2a0c9e7c54c 

```

* Apply the claim.
`kubectl apply -f spc.yaml`

* Verify the claim.
`get csp`

* Verify pool creation on appropriate nodes.
`kubectl get pod -n openebs -o wide`

* Create a StorageClass that uses the new claim.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-standard
  annotations:
    openebs.io/cas-type: cstor
    cas.openebs.io/config: |
      - name: StoragePoolClaim
        value: "cstor-disk-pool"
```
provisioner: openebs.io/provisioner-iscsi
## 1. Replacing a failed disk

* Exec into the pod associated with the failed disk.
`kubectl exec -it cstor-disk-pool-nq1i-7c686f8cb9-zbc4n -n openebs -ccstor-pool-mgmt bash`

* Use zpool to check the disk status.
`zpool status`

* Find the device node for the replacement disk.
`kubectl get bd -n openebs blockdevice-bd952ccaa0638d1a5f7ab334e65d7aad -o yaml`
* e.g.
```
...
  devlinks:
  - kind: by-id
    links:
    - /dev/disk/by-id/scsi-0Google_PersistentDisk_cstor-it-disk-10
    - /dev/disk/by-id/google-cstor-it-disk-10
...
```

* Create a BlockDeviceClaim for the new disk.
```
apiVersion: openebs.io/v1alpha1
kind: BlockDeviceClaim
metadata:
  finalizers:
  - storagepoolclaim.openebs.io/finalizer
  Labels:
     # Put the name of your SPC
    openebs.io/storage-pool-claim: cstor-disk-pool
  # Put the name of bdc as bdc-<bd-uid>
  # This 'bd-uid' is the uid of new BD that we are going to use
  # The UID can be found via 'kubectl get bd <bd-name> -o yaml' 
  name: bdc-2a1e53a2-d06e-11e9-bfdd-42010a8000a5
  # If you have installed openebs in a different namespaces then put 
  # then put that namespace here.
  namespace: openebs
  ownerReferences:
  - apiVersion: openebs.io/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: StoragePoolClaim
    # Put the name of your SPC
    name: cstor-disk-pool
    # Put the UID of your SPC
    uid: 70df195f-d08d-11e9-bfdd-42010a8000a5
Spec:
  # Put the block device name that you are going to use as a replacement
  blockDeviceName: blockdevice-bd952ccaa0638d1a5f7ab334e65d7aad 
  # This is the host name of the new block device that we are going to
  # for replacement.
  hostName: gke-cstor-it-pool-1-bdf8ee93-tv69 
  resources:
    Requests:
       # Put the capacity of BD in GB here. Disks greater than 
       # or equal to this size will match.
      storage: 100G
```

* Apply the bdc.
`kubectl apply -f bdc.yaml`

* Use zpool to replace the disk.
```
zpool replace cstor-7108b3c7-d08d-11e9-bfdd-42010a8000a5  /dev/disk/by-id/scsi-0Google_PersistentDisk_cstor-it-disk-9 /dev/disk/by-id/scsi-0Google_PersistentDisk_cstor-it-disk-10
```
where cstor-7108b3c7-d08d-11e9-bfdd-42010a8000a5 is the pool name,
/dev/disk/by-id/scsi-0Google_PersistentDisk_cstor-it-disk-9 is the
failed disk, and
/dev/disk/by-id/scsi-0Google_PersistentDisk_cstor-it-disk-10 is the
replacement.

* Monitor progress.
`zpool status`

* Remove block device entry and replace with new disk.
`kubectl edit spc cstor-disk-pool`

* Verify that the spc has the new device.
`kubectl get csp --show-labels`

* Remove device link entry and replace with new device.
`kubectl edit spc cstor-disk-pool`

* Remove the block device claim for the failed drive.
