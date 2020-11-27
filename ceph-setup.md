## Node Setup
We will use each worker node as a node in the Ceph cluster. To start we will have to make a few changes to the **cluster.yml** file:

* Node names should reflect the worker node naming scheme
* Each worker node should have a second drive available for use as a Ceph OSD. If the drives aren't at **/dev/sdb** then that should be changed in the YAML as well.

The label **role=storage-node** needs to be added to each worker node.

## Create Ceph Operator
```{bash}
oc apply -f ceph/common.yml
oc apply -f ceph/operator-openshift.yml
```