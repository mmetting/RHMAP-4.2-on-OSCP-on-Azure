# RHMAP 4.2.1 on OSCP 3.3 on Azure Setup
Installation description for running RHMAP 4.2(.1) on OSCP 3.3 on Azure

## Sizing Calculator
https://access.redhat.com/labs/rhmap/

## Prepare Azure & Install OSCP
https://drive.google.com/drive/u/0/folders/0B7XmlXDrtzYHRVgxLUw4dHVKb0U

## Prep Environment with Subscriptions
https://access.redhat.com/documentation/en/red-hat-mobile-application-platform/4.2/paged/installing-rhmap/chapter-2-preparing-infrastructure-for-installation

## Change NFS Volume for gitlab-shell pod
1. ssh to jumphost1
2. from there ssh to master1
3. `oc get pv` --> Pick a PV with 5Gi
4. `oc describe pv <PVWITH5Gi>`
5. check directory path on NFS server (jumphost1)

```
Name:	5gb-rhmap-7
Labels:	<none>
Status:	Bound
Claim:	rhmap-core/git-data
Reclaim Policy:	Recycle
Access Modes:	RWO,RWX
Capacity:	5Gi
Message:	
Source:
Type:	NFS (an NFS mount that lasts the lifetime of a pod)
Server:	jumphost1
Path:	/exports/5gb-rhmap-7
ReadOnly:	false
No events.
```

6. `oc edit pv <PVWITH5Gi>` --> Make it 6Gi


```
apiVersion: v1
kind: PersistentVolume
metadata:
annotations:
kubectl.kubernetes.io/last-applied-configuration: '{"kind":"PersistentVolume","apiVersion":"v1","metadata":{"name":"5gb-rhmap-7","selfLink":"/api/v1/persistentvolumes/5gb-rhmap-7","uid":"5f4512d3-c2a5-11e6-abca-000d3ab0d9f4","resourceVersion":"86033","creationTimestamp":"2016-12-15T09:03:40Z"},"spec":{"capacity":{"storage":"6Gi"},"nfs":{"server":"jumphost1","path":"/exports/5gb-rhmap-7"},"accessModes":["ReadWriteOnce","ReadWriteMany"],"persistentVolumeReclaimPolicy":"Recycle"},"status":{"phase":"Available"}}'
pv.kubernetes.io/bound-by-controller: "yes"
creationTimestamp: 2016-12-15T09:03:40Z
name: 5gb-rhmap-7
resourceVersion: "101212"
selfLink: /api/v1/persistentvolumes/5gb-rhmap-7
uid: 5f4512d3-c2a5-11e6-abca-000d3ab0d9f4
spec:
accessModes:
- ReadWriteOnce
- ReadWriteMany
capacity:
storage: 6Gi
claimRef:
apiVersion: v1
kind: PersistentVolumeClaim
name: git-data
namespace: rhmap-core
resourceVersion: "101205"
uid: 3906eb66-ec58-11e6-8186-000d3ab0d9f4
nfs:
path: /exports/5gb-rhmap-7
server: jumphost1
persistentVolumeReclaimPolicy: Recycle
status:
phase: Bound

```

7. then `exit` and you are back on jumphost1
8. `sudo vi /etc/exports.d/pv.vols.exports`
9. change correct line to `no_root_squash`

```
...
/exports/5gb-rhmap-7 *(rw,no_root_squash,no_wdelay)
...
```

10. save and exit
11. `exportfs -a`

## Install Core

https://access.redhat.com/documentation/en/red-hat-mobile-application-platform/4.2/paged/core-administration-and-installation-guide/chapter-1-provisioning-an-rhmap-4x-core-in-openshift-3

> Stop at 1.2.2.5. Set Up Back End Components

### Change PV definition for gitlab-shell to make it 6Gi

```
sudo vi /opt/rhmap/templates/core/fh-core-backend.json
```

```
{
"name": "GITLAB_SHELL_PVC_SIZE",
"description": "The size in bytes of the volume for repositories. Can be unadorned integers, or fixed-point integers with one of these SI suffices (E, P, T, G, M, K, m) or their power-of-two equivalents (Ei, Pi, Ti, Gi, Mi, Ki). For example, the following represent roughly the same value: 128974848, 129e6, 129M , 123Mi. Small quantities can be represented directly as decimals (e.g., 0.3), or using milli-units (e.g., 300m).",
"value": "6Gi",
"required": true
}
```

> Continue at 1.2.2.5. Set Up Back End Components
> Change millicore dc to use http for Git

## Install MBaaS
https://access.redhat.com/documentation/en/red-hat-mobile-application-platform/4.2/paged/mbaas-administration-and-installation-guide/chapter-3-provisioning-an-rhmap-4x-mbaas-in-openshift-3

> Stop at 3.4.2.2. Installation - Step 4

### Split 3-Node MBaaS Template into 3 templates
1. MongoDB

- Edit 1st MBaaS template to only use 25Gi per MongoDB node
- Run template ( [fh-mbaas-template-3node-1.json](/Users/mmetting/RedHat/tmp/OSCP/mbaas-templates/fh-mbaas-template-3node-1.json) ) deployment

```
oc new-app -f fh-mbaas-template-3node-1.json \
-p SMTP_SERVER="${SMTP_SERVER}" \
-p SMTP_USERNAME="${SMTP_USERNAME}" \
-p SMTP_PASSWORD="${SMTP_PASSWORD}" \
-p SMTP_FROM_ADDRESS="${SMTP_FROM_ADDRESS}" \
-p ADMIN_EMAIL="${ADMIN_EMAIL}"
```

> Note down generated values!

2. Mongo Initiator

- Run template ( [fh-mbaas-template-3node-2.json](/Users/mmetting/RedHat/tmp/OSCP/mbaas-templates/fh-mbaas-template-3node-2.json) ) deployment

```
oc new-app -f fh-mbaas-template-3node-2.json \
-p SMTP_SERVER="${SMTP_SERVER}" \
-p SMTP_USERNAME="${SMTP_USERNAME}" \
-p SMTP_PASSWORD="${SMTP_PASSWORD}" \
-p SMTP_FROM_ADDRESS="${SMTP_FROM_ADDRESS}" \
-p ADMIN_EMAIL="${ADMIN_EMAIL}"
```

> Edit 2nd MBaaS template to use generated values from 1st template

3. Rest of MBaaS

- Run template ( [fh-mbaas-template-3node-3.json](/Users/mmetting/RedHat/tmp/OSCP/mbaas-templates/fh-mbaas-template-3node-3.json) ) deployment

```
oc new-app -f fh-mbaas-template-3node-3.json \
-p SMTP_SERVER="${SMTP_SERVER}" \
-p SMTP_USERNAME="${SMTP_USERNAME}" \
-p SMTP_PASSWORD="${SMTP_PASSWORD}" \
-p SMTP_FROM_ADDRESS="${SMTP_FROM_ADDRESS}" \
-p ADMIN_EMAIL="${ADMIN_EMAIL}"
```

> Edit 3rd MBaaS template to use generated values from 1st template

> Continue at 3.4.3. Verifying The Installation
