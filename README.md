# workload-migration-between-clusters-on-OpenShift-Container-Platform-4

This repository provides utility playbook to help with an automated way to perform workload migration between two OpenShift Container 4.x clusters.  
Some example use case could be that you want to do a cluster wide backup and then restore to a similar cluster version or to a newer cluster, or you are using a Blue/Green cluster deployment approach and want to migrate all workloads running on the exiting green cluster to the new blue cluster so that traffic can be redirected to that cluster or you just want to take the backup and keep for situations where you might need to restore the whole cluster. 

There are several ways to go about this on OpenShift but given that Red Hat provides a few supported apporaches we will first focus on those and perhaps later add more playbooks to use the Kubernetes CSI snapshot approach wich is a little more involved that the first two apporached supported by Red Hat, and finally add a few more like using volume snapshots and reattachment approach if the platform the clusters were deployed to support that apporach. 

The two approaches to be used here are the Migration Toolkit for Containers (MTC) and the OpenShift API for Data Protection (OADP). 
Refer to the official documentaion for more information on [MTC](https://docs.openshift.com/container-platform/4.14/migration_toolkit_for_containers/about-mtc.html).
Refer to the official documentaion for more information on [OADP](https://docs.openshift.com/container-platform/4.14/backup_and_restore/application_backup_and_restore/oadp-intro.html).

Note that both apporaches use the community velero project on which you can find more information [here](https://velero.io/) and [here](https://velero.io/docs/v1.15/).

The key difference between the approaches is that even though they both use an S3 like storage location to temporarily store the kubernetes objects retrieved from the target cluster to be used later to during the restore to the destination cluster, MTC does both steps in one transaction whereas OADP performs them separately; namely backups are performed separately and then once successful the restore operation can be started using the backup. 
Another key difference is that MTC also has an option for direct migration, which skips the transient step of storing the items in the S3 bucket and therefore is even faster assuming that you don't require the control or the backup so it can later be used. There is also a live migration option that makes MTC even faster but has more stringent requirements. 

So the playbooks in this repository are organized around the workflow for each of those approaches in the initial release. 

To perform an MTC migration you would simply use the `post-deploy-configure-mtc-migration.yml` playbook assuming you have the list of namespaces and all of the other required variables. In case you want to dynamically retrieve the namespace list from the source cluster as well as the SA token required to configure the MTC migration, you can run the `mtc-targeted-namespace-list-retrieval.yml` playbook to get the namespace list and the SA Token set as fact that will later be used by the first playbook referenced above. or you can set the appropriate variable for the main playbook to fetch that information for you.  In case you don't already have the S3 bucket created on an ODF cluster you can use the provision-backup-s3-bucket-on-odf.yml playbook to create the OBC and retrieve the related credentials. Otherwise, use the retrieve-backup-s3-bucket-on-odf.yml to fetch the S3 bucket information from ODF.

Finally, note that given that MTC process is one step it will be run from the destination cluster and if necessary will need ome information to be able to connect to the source clustr to fetch the necessary infromation if not provided.

In contrast and as mentioned above the OADP flow is a two steps process perfromed on two different clusters, where backups are taken in the first step (on the source cluster) then the restore is performed in the second step upon a successful completion of the backup (on the destination cluster). 
There are sereval ways to perform an OADP backup. For example if you have RHACM (Red Hat Advanced Cluster Management) you can use an ACM policy, for which you get a starting OADP Backup policies from the [community repository](https://github.com/open-cluster-manager-io/policy-collection) and then adjust them to suit your needs if necessary. Another way to deploy OADP to perform the backup is to use ansible and here will be providing some helper playbooks that can be used to setup the required OADP CRs so that a backup can be performed on a schedule or ad'hoc depending on your preference. 

Similarly an OADP restore can be done using various approaches like using a playbook to run it, using an ACM policy to trigger the restore or manually applying the OADP restore related CRs to the destination cluster. 
Various helper playbooks are  provided here to help check prerequisites on the destination cluster as well as configure any necessary objects required for a successful restore like ensuring that snapshot class exist as well as storage classes and any other required kunernetes manifests. 

Another simple approach might be to download all existing persistence objects (PVs and PVCs) in the current cluster and then reapply them in the new cluster. This apporach even though simplistic would only work if the object backing those persistent objects reside in the same region and availability zone and can easily be rebound/attached by the nodes in the new cluster. If you would rather follow that at your own risk there are playbooks that will help you download the persistence related manifests.

## MTC Cluster Workload Migration Flow
A quick recap:
Use the provided playbooks to perform an MTC migration. 
As already mentioned above MTC is a one step process for the migration transaction which is run from the destination cluster, even though the provided playbook can be run several times or at least twice, once to setup the MTC operator and related CRs and once to trigger the migration by applying the appropriate CR to kick off the migration. To perform an MTC migration use the provided playbooks as follows:
1. Ensure the source cluster meets the required prerequisites before starting the process. Run the provided `validate-source-cluster-pre-backup.yml`playbook to ensure the source cluster  is ready for the migration.
```
ansible-playbook --ask-vault-pass  -vvv validate-source-cluster-pre-backup.yml
```
2. Ensure the destination cluster also meets the required prerequisites before starting the process. Run the provided `validate-destination-cluster-info-pre-restore.yml` playbook to ensure the cluster is ready for the migration.
```
ansible-playbook --ask-vault-pass  -vvv validate-destination-cluster-info-pre-restore.yml
```
3. Review the results of the previous steps to ensure that the clusters are ready for the migration. 
4. Ensure you have the storage location bucket information readily available to use. If you don't already have a bucket use the provided `provision-backup-s3-bucket-on-odf.yml` playbook to set an OBC and bucket on ODF as well and retrieve the associated information. Otherwise use the `retrieve-backup-s3-bucket-on-odf.yml` playbook to fetch the S3 bucket information from ODF. Run either of the playbooks as follows:
```
ansible-playbook --ask-vault-pass  -vvv provision-backup-s3-bucket-on-odf.yml 
```
or 
```
ansible-playbook --ask-vault-pass  -vvv retrieve-backup-s3-bucket-on-odf.yml
```
5. Perform the MTC migration by running the provided `post-deploy-configure-mtc-migration.yml` playbook. This playbook can be run in several ways. For example if you don't have the required information from the source cluster (e.g. SA token, targeted namespace list) you set the appropriate variables for this playbook to fetch that information and use it to configure MTC. Also it can be used to configure the necessary MTC components (CRs) up to the migration plan without triggering the migration and then run again to perform the migration. It is recommended to run the playbook to deploy and configure MTC like described above by running the playbook like below:
```
ansible-playbook --ask-vault-pass  -vvv -e retrieve_satoken=false -e add_mig_cluster=true -e add_replication_repo=true -e add_migration_plan=true -e perform_migration=false  -e src_mtc_cluster_console_url=<src-cluster-api> -e skip_api_login_logout=true  post-deploy-configure-mtc-migration.yml
```
Once the migration plan CR is ready you can either perfrom the migration through the web console or rerun the playbook again using the following variable to trigger the migration.
```
ansible-playbook --ask-vault-pass  -vvv -e retrieve_satoken=false -e add_mig_cluster=false -e add_replication_repo=false -e add_migration_plan=false -e perform_migration=true -e src_mtc_cluster_console_url=<src-cluster-api> -e skip_api_login_logout=true  post-deploy-configure-mtc-migration.yml
```


## OADP Cluster Workload Migration Flow
A quick recap:
1. Use the provided playbooks to perform an OADP backup
2. Use the provided playbooks to perform an OADP restore upon successful completion of the backup step above.

### OADP Cluster Workload Backup 
To backup the cluster workload using OADP you need to ensure that the source cluster meets the requirements for an OADP backup adnd use the provided playbooks to perform the backup as follows:

1. Ensure you have the storage location bucket information readily available to use. If you don't already have a bucket use the provided `provision-backup-s3-bucket-on-odf.yml` playbook to create an OBC and bucket on ODF as well and retrieve the associated information. Otherwise, use the `retrieve-backup-s3-bucket-on-odf.yml` playbook to fetch the S3 bucket information from ODF. Run either of the playbooks as follows:
> [!WARNING]
> This playbook should only be run against the cluster hosting the ODF S3 not the target backup cluster. 
```
ansible-playbook --ask-vault-pass  -vvv provision-backup-s3-bucket-on-odf.yml 
```
or 
```
ansible-playbook --ask-vault-pass  -vvv retrieve-backup-s3-bucket-on-odf.yml
```
> [!NOTE]
> The following steps are to be performed against the targeted backup cluster.  
2. Ensure your source cluster meets all the prequisites before getting started with the OADP backup. Use the provided validation playbook (`validate-source-cluster-pre-backup.yml`) to perform the validation checks as follows:
```
ansible-playbook --ask-vault-pass  -vvv validate-source-cluster-pre-backup.yml
```
3. Review the output of the previous step and ensure any necessary steps are taken to remediate any failures.
4. Optional step, If you have alist of namespace to use or if you want to make further modifications to the list retrieved by the provided playbook, then ensure you have an updated list of targeted namespaces. 
```
ansible-playbook --ask-vault-pass  -vvv  migration-targeted-namespace-list-retrieval.yml
```
5. Ensure there are no objects in invalid state that can cause the backups to fail. For example invalid imagestreams or pending PVCs can cause failures. Use the `process-oadp-exclusion-on-source-cluster-pre-backup.yml`playbook to excluse any objects in the cluster that meets that requirement. For the time being the provided playbook only excludes unbound PVCs and imagestreams with certain error conditions, non running pods, complete jobs but can be updated to add more resources to be excluded. Run the playbook as follows:
```
ansible-playbook --ask-vault-pass  -vvv  process-oadp-exclusion-on-source-cluster-pre-backup.yml
```
> [!NOTE]
> There is a cronjob template for this process that can be used if you are using a schedule CR with yous backup or if you are using an ACM policy that has a schedule object in order to run this on a predefined schedule before the backup cron is triggered (e.g run this 20 min before the backup is run). The cronjob has all the necessary data, objects and associted configmaps and all can easily be used to either create a new ACM policy or added as ObjectDefinition templates within the existing ACM policy you are already using for your backup.

6. Perform the OADP backup by running the provided `post-deploy-configure-oadp-migration-backup.yml` playbook. Run the playbook as follows:
```
ansible-playbook --ask-vault-pass  -vvv  post-deploy-configure-oadp-migration-backup.yml
```
7. Validate that the cluster backup is successful by either visually reviewing the status of the velero backup CR created in the step above upon completion of the backup or by using the provided `validate-source-cluster-post-backup.yml` playbook to check that all of the necessary velero CR objects were created with the appropriate status. Run the playbook as follows:
```
ansible-playbook --ask-vault-pass  -vvv validate-source-cluster-post-backup.yml
```
8. Optional steps. In the event of failures use the provided helper playbooks to perform resources cleanup. For examle if you need to cleanup the OADP deployment you can use the `undeploy-oadp-migration-application.yml`. Similarly, if you need to remove failed backup CRs you can run the `remove-failed-backup-from-source-cluster.yml` playbook.

Note that to successfully run each of the playboks referenced above it is important to either update the appropriate variables either through the oadp-migration.yml variable file or by passing the variables as extra argument to the `ansible-playbook`commands.

> [!TIP]
> Given that OADP backup using the CSI snapshot and Data Mover only works for workloads with bound PVCs, a set of tasks has been added to do download worload related manifests for all the namespace that were not selected for backup so that if necessary one can use those manifests to recreate the workload related kubernetes object in the new cluster.


### OADP Cluster Workload Restore
To restore the workload to the destination cluster run the provided playbooks as follows:  
1. Ensure that there is succesful backup.
> [!NOTE]
> The following step is to be performed against the backup cluster.  
```
ansible-playbook --ask-vault-pass  -vvv validate-source-cluster-pre-restore.yml 
```
> [!NOTE]
> The following steps are to be performed against the targeted restore/destination cluster.  
2. Ensure the destination cluster meets the prerequisite to be used for an OADP restore. To do this use the provided playbook to perfrom the prerequisite validation.
```
ansible-playbook --ask-vault-pass  -vvv validate-destination-cluster-info-pre-restore.yml
```
3. Review the result of the previous step and ensure the cluster has the necessary resources for the OADP restore phase.
4. Perform the OADP restore by running the provided restore playbook as follows:
```
ansible-playbook --ask-vault-pass  -vvv post-deploy-configure-oadp-migration-restore.yml
```
5. Optional step. If necessary you can use the provided playbook to download and apply raw manifests that were created during the backup step for namespaces not matching the strict requirement for an OADP backup so that you can apply those manifests to the the restore cluster if you want to. You can use the `post-deploy-configure-apply-raw-manifests-to-destination-cluster.yml` playbook to apply those raw manifests to the restore cluster. 
6. Optional steps. In the event of failures use the provided helper playbooks to perform resources cleanup. For examle if you need to cleanup the OADP deployment you can use the `undeploy-oadp-migration-application.yml`. 

Note that for each of the playbook steps above you need to ensure the appropriate variables are provided either through the CLI using the -e option for the `ansible-playbook` command or by updating the `oadp-migration.yml` variable file.


## Raw Persistence manifests Cluster Workload Migration Flow
In this flow, the playbook will identify all bound persistent volume claims and related volumes and download the manifests and perform metadata cleanup of the downloaded manifests so that they can readily be applied to the new cluster. Note that there is no garantee that the baking volumes can be easily attached and bound by nodes in the new cluster. 

### Raw Persistence manifests Workload Backup 
To backup the cluster workload using the Raw persistence manifest download, you need to ensure that the source cluster meets the requirements for manifests backup and use the provided playbooks to perform the backup as follows:
1. Ensure your  source cluster meets all the prequisites before getting started with the raw manifests backup. Use the provided validation playbook (`validate-source-cluster-pre-backup.yml`) to perform the validation checks as follows:
```
ansible-playbook --ask-vault-pass  -vvv validate-source-cluster-pre-backup.yml
```
2. Review the output of the previous step and ensure any necessary steps are taken to remediate any failures.
3. Ensure you have an updated list of targeted namespaces. If you have captured the list some time ago ago, ensure it is still accurate as an incorrect list could cause the process to fail. Use the provided playbook (`migration-targeted-namespace-list-retrieval.yml`) to retrieve the updated list of targeted namespaces from the source cluster.
```
ansible-playbook --ask-vault-pass  -vvv  migration-targeted-namespace-list-retrieval.yml
```
4. Perform the raw backup by running the provided `post-deploy-configure-raw-pvc-download-from-source-cluster.yml` playbook. Run the playbook as follows:
```
ansible-playbook --ask-vault-pass  -vvv post-deploy-configure-raw-pvc-download-from-source-cluster.yml 
```

### Raw Persistence manifests Cluster Workload Restore
To restore the workload to the destination cluster, apply all of the downlaoded manifests to the new cluster (ensuring the PVs are applied first and the the PVCs) using the oc command like `oc apply -f <pvs-dir>` and `oc apply -f <pvc-dir`. Validate that the objects were properly applied and that the backing volumes were properly mounted by the nodes in the new cluster. Also verify and ensure that the nodes on which the volumes were attached are also the nodes on which the workloads that need those volumes are scheduled on.   
If necessary use the provided `post-deploy-apply-raw-manifests-to-destination-cluster.yml` playbook to help dwonload and apply the raw manifests to the restore cluster. You can do so using the following example.  
```
ansible-playbook --ask-vault-pass  -vvv post-deploy-apply-raw-manifests-to-destination-cluster.yml 
```



Playbook Variables
--------------

| migration approach        |      Variable file              | 
|---------------------------|---------------------------------|
|  MTC                      |   mtc-migration.yml             |
|  OADP                     |   oadp-migration.yml            |
|  Raw                      |   oadp-migration.yml            |


Update the appropriate file for you migration approach and ensure you have all variables set before begenning the process. 

There are few other optional variables that you can change to your liking but don't have to change. 


Dependencies
------------

A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles.


Installation and Usage
-----------------------
Clone the repository to where you want to run this from and make sure you can connect to your cluster using the CLI . 
You will also need to have cluster-admin role in order to run the playbook .
Finally before running the playbook make sure to set and update the variables as appropriate to your use case.

Playbooks
---------

To run the MTC  based playbook use the ansible-playbook command as follows   
`ansible-playbook  post-deploy-configure-mtc-migration.yml --ask-vault-pass -vvv`  
The above playbook performs the backup, restore , while optionally uploading the various items into S3 if necessary.
Use the following command to run the optional namespace list playbook to get a dynamic list of namespace to include in the backup and restore operations of the the MTC migration.
`ansible-playbook mtc-targeted-namespace-list-retrieval.yml --ask-vault-pass -vvv`  

To run the ODAP related playbooks use the ansible-playbook command as follows   
`ansible-playbook  validate-source-cluster-pre-backup.yml --ask-vault-pass -vvv`  
The above playbook ensures that the source cluster is ready for an OADP backup and performs any prerequisite tasks related to the backup operation (e.g. check if there are any failing pods, any pods with too many restarts, any pending pvcs, any pending csrs, any failing imagestreams ...) . 

`ansible-playbook  validate-destination-cluster-info-pre-restore.yml --ask-vault-pass -vvv`  
The above playbook ensures that the destination cluster is ready for an OADP restore and performs any prerequisite tasks related to the restore operation (e.g. check if the volume snapshot class is properly configured, namespaces are properly defined if necessary, node count matches what is required for the target workload from th source cluster, cluster and machines autoscaler are defined and configured if necessary, all requried storage classes are configured as necessary from the source cluster ...) . 


License
-------

BSD

Author Information
------------------

