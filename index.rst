##################################
Rebuilding GKE Kubernetes clusters
##################################

.. abstract::

   A runbook for destroying and recreating GKE Kubernetes clusters with no data loss and minimal downtime.


Sometimes we need to make changes to a `GKE`_ Kubernetes cluster in Google Cloud that requires us to destroy and rebuild the cluster from scratch.
One such change is `enabling GKE Dataplan V2`_.
This technote describes a way to destroy and recreate the cluster with no data loss and minimal downtime.

Prerequisites
=============

Backup for GKE
--------------

Make sure that the `Backup for GKE`_ API is enabled.
This command will exit non-zero if it is not:

.. code-block:: console

   $ PROJECT=roundtable-dev-abe2 gcloud services list --enabled --project $PROJECT | grep gkebackup.googleapis.com
   gkebackup.googleapis.com                Backup for GKE API

Make sure that the `Backup for GKE`_ cluster addon is enabled in the cluster you are rebuilding.
This command will exit non-zero if it is not:

.. code-block:: console

   $ CLUSTER_NAME=roundtable-dev \
     PROJECT_ID=roundtable-dev-abe2 \
     LOCATION=us-central1 \
     gcloud container clusters describe $CLUSTER_NAME \
       --project $PROJECT_ID \
       --location $LOCATION \
       --format json \
       | jq -e .addonsConfig.gkeBackupAgentConfig.enabled

The best way to enable these things if they are not enabled is to make a PR to the `idf_deploy`_ Terraform config repo and merge it.
That way, we won't forget to enable it if the environment needs to be recreated.

Static IPs
----------

Every GKE cluster has an instance of `ingress-nginx`_ running in it.
The Helm chart that installs it creates a ``LoadBalancer`` ``Service`` that provisions a Google Cloud load balancer to recieve traffic from outside the cluster.
This load balancer has an IP address attached to it, and we create a DNS record (in `AWS Route53`_ for now) that points to it.
This IP address is set in Phalanx as the `loadBalancerIP` value in the `ingress-nginx` app.
It is important not to change this IP addresses when we recreate the cluster, or else we would also have to update the DNS records.

By default, this is an `ephemeral IP address`_, which means that it will disappear when the Google Cloud load balancer gets destroyed.
The load balancer might get destroyed when we destroy the cluster to rebuild it.
This IP address should be a `Google Cloud static external IP addresses`_.
The best way to ensure this is a static and not ephemeral IP address is to make sure it is provisioned in our `idf_deploy`_ Terraform config repo.

There may be other static IP addresses that we depend on:

* External Kafka broker access
* External InfluxDB access

You should ensure these are also static, and not ephemeral, IP addresses in Google Cloud.

If some external IPs are ephemeral, and not static, in Google Cloud, you will probably have to update a DNS ``A`` record when the new IP gets provisioned.
You should then add config to the `idf_deploy`_ repo for that IP address and import it into the Terraform state so it doesn't get destroyed next time.

ArgoCD sync state
------------------

Make sure that every app in ArgoCD is in sync before starting.
That way, you can be sure that the ArgoCD state in the backup you're about to take is the actual preferred state of the cluster.
You can be sure that out of sync in the new restored cluster is because of the restore process.

Terraform plan
--------------

Make sure that the terraform plan for the cluster that you are rebuilding shows no changes.
That way, you can be sure that the only changes that you make during this process are intentional.
You don't want to make old changes that were never applied, or revert any manual changes, at the same time as you are doing this cluster restore operation.

Runbook
=======

Make your change in idf_deploy
------------------------------

All changes should be made in the `idf_deploy`_ repo, so the first step is to make a PR to `idf_deploy`_ that contains your change, and make sure the Terraform plan looks good.

Announce downtime and data loss
-------------------------------

You're about to create a complete backup of the persistent volumes and Kubernetes objects in the cluster.
Any changes to data on persistent volumes or Kubernetes objects in the cluster after the backup is made will not be restored into the new cluster.
Make sure you notify all of the necessary people of this fact.

Create a backup of the cluster
------------------------------

Take an on-demand backup of the cluster using `Backup for GKE`_.
Every cluster should have a backup plan with the same name as the cluster.
Create an on-demand backup using that plan through the web console UI, or by using a command like this:

.. code-block:: console

   $ PROJECT_ID=roundtable-dev-abe2 \
       LOCATION=us-central1 \
       BACKUP_PLAN=roundtable-dev \
       BACKUP_NAME=before-rebuild \
       gcloud beta container backup-restore backups create $BACKUP_NAME \
         --project $PROJECT_ID \
         --location $LOCATION \
         --backup-plan $BACKUP_PLAN \
         --wait-for-completion

   $ gcloud

This will take several minutes.
If you don't want to have the command wait, omit the ``--wait-for-completion`` option.
See the `on-demand backup docs`_ for more options.


Manually delete the existing cluster
------------------------------------

The terraform module we're using tries to create the new cluster before destroying the old cluster.
This won't work because we want to keep the name of the new cluster the same, so we need to delete the cluster manually.

#. Manually delete all of the Services of type ``LoadBalancer`` in the cluster. The `GKE cluster deletion docs`_ recommend doing this to ensure that the associated Google Cloud load balancer instances are deleted.
#. Delete the cluster through the Google Cloud web console UI or with this command::

     $ CLUSTER_NAME=roundtable-dev \
       PROJECT_ID=roundtable-dev-abe2 \
       LOCATION=us-central1 \
       gcloud container clusters delete $CLUSTER_NAME --project $PROJECT_ID --location $LOCATION

   This will take several minutes.


Merge the ``idf_deploy`` PR
---------------------------

This will create a new cluster, including the changes that you made that required destroying and recreating the cluster.
This will take several minutes

Regenerate local Kubernetes API creds
-------------------------------------

Follow the directions in the `Phalanx environments page`_ for this cluster to regenerate local API credentials so you can run commands with ``kubectl``.
Something like this:

.. code-block:: console

   $ CLUSTER=roundtable-dev \
     PROJECT_ID=roundtable-dev-abe2 \
     gcloud container clusters get-credentials $CLUSTER --project $PROJECT_ID --region us-central

Restore the backup
------------------

Create a restore of the on-demand backup of the cluster using `Backup for GKE`_.
Every cluster should have a restore plan with the same name as the cluster.
Create restore using that plan through the web console UI, or by using a command like this:

.. code-block:: console

   $ echo '{"exclusionFilters": [{"groupKind": {"resourceGroup": "acme.cert-manager.io", "resourceKind": "Order"}}, {"groupKind": {"resourceGroup": "acme.cert-manager.io", "resourceKind": "Challenge"}}, {"groupKind": {"resourceGroup": "cert-manager.io", "resourceKind": "CertificateRequest"}}]}' > /tmp/gke-backup-filter-file.json && \
     PROJECT_ID=roundtable-dev-abe2 \
     RESTORE_NAME=after-rebuild \
     LOCATION=us-central1 \
     RESTORE_PLAN=roundtable-dev \
     BACKUP_PLAN_ID=roundtable-dev \
     BACKUP_ID=before-rebuild \
     gcloud beta container backup-restore restores create $RESTORE_NAME \
       --project $PROJECT_ID \
       --location $LOCATION \
       --restore-plan $RESTORE_PLAN \
       --filter-file /tmp/gke-backup-filter-file.json \
       --backup projects/$PROJECT_ID/locations/$LOCATION/backupPlans/$BACKUP_PLAN_ID/backups/$BACKUP_ID

This will take several minutes.
You can view the progress of the restore in the Google Cloud web console UI.

We need to explicitly exclude certain namespace-scoped `cert-manager`_ resources (see the `cert-manager backup docs`_ for details).
Unfortunately, namespace-scoped resources can not be excluded in a restore plan, so we have to exclude them when we're creating the restore itself.

When the backup is completely restored, you should be able to access the `ArgoCD`_ instance for the new cluster.


Fix cert-manager
----------------

`Backup for GKE`_ doesn't restore resources into certain `managed namespaces`_, including ``kube-system``.
`cert-manager`_ installs some ``Role``s into the ``kube-system`` namespace, so we have to manually sync those roles into the cluster.
Do this in ArgoCD by syncing the `cert-manager app`_.

Fix Sasquatch
-------------

When a Kafka cluster is created with a `Strimzi`_ ``Kafka`` CRD, it gets assigned a random ID.
The data in our backed-up persistent volumes will contain a different ID, and the Kafka broker pods will not be able to start because of this.
You need to manually change the Strimzi Kafka cluster ID to match the cluster ID in the persistent volume.
See this `Strimzi discussion about the ID mismatch`_, and the `Strimzi docs for pausing reconciliation`_ for more information.

#. Look in the logs of the failing kafka pods.
   There should be a message that says something like "Exception in thread "main" java.lang.RuntimeException: Invalid cluster.id in: /var/lib/kafka/data-0/kafka-log0/meta.properties. Expected <new-cluster-ID>, but read <old-cluster-ID>".
   Note the old cluster ID in that message.
#. Pause the Strimzi reconciliation of the ``Kafka`` object by adding an annotation::

       $ CONTEXT=roundtable-dev \
         kubectl --context $CONTEXT --namespace sasquatch \
            annotate Kafka sasquatch strimzi.io/pause-reconciliation="true"
#. Edit the `clusterID` in the **status** of the Sasquatch ``KafkaNodePool`` controler resource::

       $ CONTEXT=roundtable-dev \
         kubectl --context $CONTEXT --namespace sasquatch \
            patch KafkaNodePool controller \
            --type=merge --subresource status --patch 'status: {clusterId: old-cluster-id}'
#. Resume Strimzi reconciliation by removing the pause annotation::

       $ CONTEXT=roundtable-dev \
         kubectl --context $CONTEXT --namespace sasquatch \
           annotate Kafka sasquatch strimzi.io/pause-reconciliation-
#. Wait for all resources in the sasquatch app to stabilize.
   This may take several minutes.

Sync and prune apps in ArgoCD
-----------------------------

Some apps may show as out of sync in ArgoCD.
This is usually because certain resources that get restored from the old cluster have invalid references to the custom resources that created them, because those creator resources will have different UIDs in the restored cluster.
In these cases, the created resources will show up as needing to be deleted in ArgoCD.
Some examples:

* ``Secret`` created by ``VaultSercret``
* ``Secret`` created by ``GafaelfawrServiceToken``
* ``Certificate`` created by ``cert-manager``-annotated ``Ingress``.

In these cases, it is OK to sync and prune these resources, because they will just get created again.

.. warning::

   Be very careful when syncing and pruning apps.
   Carefully inspect each change and make sure you know that the resources that will get pruned will get recreated by some controller.
   Make sure you understand these and all other changes that show up in the ArgoCD diffs.
   If there is something you don't understand, get some help and figure it out before you sync and prune!


.. _AWS Route53: https://aws.amazon.com/route53/
.. _ArgoCD: https://argo-cd.readthedocs.io/en/stable/
.. _Backup for GKE: https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/concepts/backup-for-gke
.. _GKE cluster deletion docs: https://docs.cloud.google.com/kubernetes-engine/docs/how-to/deleting-a-cluster
.. _GKE: https://cloud.google.com/kubernetes-engine
.. _Google Cloud static external IP addresses: https://cloud.google.com/compute/docs/ip-addresses/configure-static-external-ip-address
.. _Phalanx environments page: https://phalanx.lsst.io/environments/index.html
.. _Strimzi discussion about the ID mismatch: https://github.com/orgs/strimzi/discussions/10082
.. _Strimzi docs for pausing reconciliation: https://strimzi.io/docs/operators/latest/full/deploying#proc-pausing-reconciliation-str
.. _Strimzi: https://strimzi.io/
.. _cert-manager app: https://phalanx.lsst.io/applications/cert-manager/index.html
.. _cert-manager backup docs: https://cert-manager.io/docs/devops-tips/backup
.. _cert-manager: https://cert-manager.io/
.. _enabling GKE Dataplan V2: https://docs.cloud.google.com/kubernetes-engine/docs/concepts/dataplane-v2
.. _ephemeral IP address: https://cloud.google.com/vpc/docs/ip-addresses#ephemeral_and_static_ip_addresses
.. _idf_deploy: https://github.com/lsst/idf_deploy
.. _ingress-nginx: https://github.com/kubernetes/ingress-nginx
.. _managed namespaces: https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/how-to/restore-plan#managed-namespaces
.. _on-demand backup docs: https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/how-to/backup
