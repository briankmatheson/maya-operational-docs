# Replace a Failed cStor Disk

## 0. Creating a cStor Pool
Show the disks.
`kubectl get bd -n openebs --show-labels`

Create a claim.
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

Apply the claim.
`kubectl apply -f spc.yaml`

Verify the claim.
`get csp`

Verify pool creation on appropriate nodes.
`kubectl get pod -n openebs -o wide`

Create a StorageClass that uses the new claim.
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

Exec into the pod associated with the failed disk.
`kubectl exec -it cstor-disk-pool-nq1i-7c686f8cb9-zbc4n -n openebs -ccstor-pool-mgmt bash`

Use zpool to check the disk status.
`zpool status`

Find the device node for the failed disk.
`kubectl get bd -n openebs blockdevice-bd952ccaa0638d1a5f7ab334e65d7aad -o yaml`
e.g.
```
...
  devlinks:

  - kind: by-id

    links:

    - /dev/disk/by-id/scsi-0Google_PersistentDisk_cstor-it-disk-10

    - /dev/disk/by-id/google-cstor-it-disk-10
...
```
