# Configure storage for backups

You can configure storage for backups in the `backup.storages` subsection of the
Custom Resource, using the [deploy/cr.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/cr.yaml)
configuration file.

You should also create the [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/)
object with credentials needed to access the storage.

=== "Amazon S3 or S3-compatible storage"

    1. To store backups on the Amazon S3, you need to create a Secret with
        the following values:
    
        * the `metadata.name` key is the name which you wll further use to refer
            your Kubernetes Secret,
        * the `data.AWS_ACCESS_KEY_ID` and `data.AWS_SECRET_ACCESS_KEY` keys are
            base64-encoded credentials used to access the storage (obviously these
            keys should contain proper values to make the access possible).

        Create the Secrets file with these base64-encoded keys following the
        [deploy/backup-secret-s3.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/backup/backup-secret-s3.yaml)
        example:

        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: my-cluster-name-backup-s3
        type: Opaque
        data:
          AWS_ACCESS_KEY_ID: UkVQTEFDRS1XSVRILUFXUy1BQ0NFU1MtS0VZ
          AWS_SECRET_ACCESS_KEY: UkVQTEFDRS1XSVRILUFXUy1TRUNSRVQtS0VZ
        ```

        !!! note

            You can use the following command to get a base64-encoded string from a plain text one:

            === "in Linux"

                ``` {.bash data-prompt="$" }
                $ echo -n 'plain-text-string' | base64 --wrap=0
                ```

            === "in macOS"

                ``` {.bash data-prompt="$" }
                $ echo -n 'plain-text-string' | base64
                ```

        Once the editing is over, create the Kubernetes Secret object as follows:

        ``` {.bash data-prompt="$" }
        $ kubectl apply -f deploy/backup-secret-s3.yaml
        ```

        !!! note

            In case if the previous backup attempt fails (because of a temporary
            networking problem, etc.) the backup job tries to delete the unsuccessful
            backup leftovers first, and then makes a retry. Therefore there will be no
            backup retry without [DELETE permissions to the objects in the bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-with-s3-actions.html).
            Also, setting [Google Cloud Storage Retention Period](https://cloud.google.com/storage/docs/bucket-lock)
            can cause a similar problem.


    2. Put the data needed to access the S3-compatible cloud into the
        `backup.storages` subsection of the Custom Resource.
    
        * `storages.<NAME>.type` should be set to `s3` (substitute the <NAME>
           part with some arbitrary name you will later use to refer this
           storage when making backups and restores).
    
        * `storages.<NAME>.s3.credentialsSecret` key should be set to the name
            used to refer your Kubernetes Secret (`my-cluster-name-backup-s3` in
            the last example).
    
        * `storages.<NAME>.s3.bucket` and `storages.<NAME>.s3.region` should
           contain the S3 bucket and region. Also you can use
           `storages.<NAME>.s3.prefix` option to specify the path (sub-folder)
           to the backups inside the S3 bucket. If prefix is not set, backups
           are stored in the root directory.
    
        * if you use some S3-compatible storage instead of the original Amazon
            S3, add the [endpointURL](https://docs.min.io/docs/aws-cli-with-minio.html)
            key in the `s3` subsection, which should point to the actual cloud
            used for backups. This value is specific to the cloud provider. For
            example, using [Google Cloud](https://cloud.google.com) involves the
            [following](https://storage.googleapis.com) endpointUrl:

            ```yaml
            endpointUrl: https://storage.googleapis.com
            ```

        The options within the `storages.<NAME>.s3` subsection are further
        explained in the [Operator Custom Resource options](operator.md#operator-backup-section).

        Here is an example of the [deploy/cr.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/cr.yaml)
        configuration file which configures Amazon S3 storage for backups:

        ```yaml
        ...
        backup:
          ...
          storages:
            s3-us-west:
              type: s3
              s3:
                bucket: S3-BACKUP-BUCKET-NAME-HERE
                region: us-west-2
                credentialsSecret: my-cluster-name-backup-s3
          ...
        ```
        
        ??? note "Using AWS EC2 instances for backups makes it possible to automate access to AWS S3 buckets based on [IAM roles](https://kubernetes-on-aws.readthedocs.io/en/latest/user-guide/iam-roles.html) for Service Accounts with no need to specify the S3 credentials explicitly."

            Following steps are needed to turn this feature on:

            * Create the [IAM instance profile](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
                and the permission policy within where you specify the access level that
                grants the access to S3 buckets.
            * Attach the IAM profile to an EC2 instance.
            * Configure an S3 storage bucket and verify the connection from the EC2
                instance to it.
            * Do not provide `s3.credentialsSecret` for the storage in `deploy/cr.yaml`.

=== "Microsoft Azure Blob storage"

    1. To store backups on the Azure Blob storage, you need to create a
        Secret with the following values:

        * the `metadata.name` key is the name which you wll further use to refer
            your Kubernetes Secret,
        * the `data.AZURE_STORAGE_ACCOUNT_NAME` and `data.AZURE_STORAGE_ACCOUNT_KEY`
            keys are base64-encoded credentials used to access the storage
            (obviously these keys should contain proper values to make the access
            possible).

        Create the Secrets file with these base64-encoded keys following the
        [deploy/backup/backup-secret-azure.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/backup/backup-secret-azure.yaml) example:

        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: azure-secret
        type: Opaque
        data:
          AZURE_STORAGE_ACCOUNT_NAME: UkVQTEFDRS1XSVRILUFXUy1BQ0NFU1MtS0VZ
          AZURE_STORAGE_ACCOUNT_KEY: UkVQTEFDRS1XSVRILUFXUy1TRUNSRVQtS0VZ
        ```

        !!! note

            You can use the following command to get a base64-encoded string from a plain text one:

            === "in Linux"

                ``` {.bash data-prompt="$" }
                $ echo -n 'plain-text-string' | base64 --wrap=0
                ```

            === "in macOS"

                ``` {.bash data-prompt="$" }
                $ echo -n 'plain-text-string' | base64
                ```

        Once the editing is over, create the Kubernetes Secret object as follows:

        ``` {.bash data-prompt="$" }
        $ kubectl apply -f deploy/backup-secret-azure.yaml
        ```

    2. Put the data needed to access the Azure Blob storage into the
        `backup.storages` subsection of the Custom Resource.
    
        * `storages.<NAME>.type should be set to `azure` (substitute the <NAME> part
           with some arbitrary name you will later use to refer this storage when
           making backups and restores).
    
        * `storages.<NAME>.azure.credentialsSecret` key should be set to the name used
            to refer your Kubernetes Secret (`azure-secret` in the last
            example).
    
        * `storages.<NAME>.azure.container` option should contain the name of the
           Azure [container](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction#containers).

        The options within the `storages.<NAME>.azure` subsection are further explained
        in the [Operator Custom Resource options](operator.md#operator-backup-section).

        Here is an example
        of the [deploy/cr.yaml](https://github.com/percona/percona-server-mongodb-operator/blob/main/deploy/cr.yaml)
        configuration file which configures Azure Blob storage for backups:

        ```yaml
        ...
        backup:
          ...
          storages:
            azure-blob:
              type: azure
              azure:
                container: <your-container-name>
                credentialsSecret: azure-secret
              ...
        ```

=== "Persistent Volume"

    Here is an example of the `deploy/cr.yaml` backup section fragment,
    which configures a private volume for filesystem-type storage:

    ```yaml
    ...
    backup:
      ...
      storages:
        fs-pvc:
          type: filesystem
          volume:
            persistentVolumeClaim:
              accessModes: [ "ReadWriteOnce" ]
              resources:
                requests:
          storage: 6Gi
      ...
    ```

    !!! note

        Please take into account that 6Gi storage size specified in this
        example may be insufficient for the real-life setups; consider using
        tens or hundreds of gigabytes. Also, you can edit this option later,
        and changes will take effect after applying the updated
            `deploy/cr.yaml` file with `kubectl`.
