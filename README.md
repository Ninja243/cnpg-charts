# cnpg

This chart is an amalgamation of these repos:
 - https://github.com/cloudnative-pg/charts (base)
 - https://github.com/pha91/charts/tree/feat/cluster-plugin-support 
 - https://github.com/pha91/charts/tree/feat/additional-env

This chart should be replaced by the base cloudnative-pg/charts repo, once 
[this PR is approved and merged](https://github.com/cloudnative-pg/charts/pull/680) which hopefully
won't take too long.

For most options refer to the [docs here](https://github.com/cloudnative-pg/charts/blob/main/README.md)

## How to use this

Merging @pha91's repos with this means we can now use the ObjectStores to define buckets to back-up
to and restore from. 

Here's how an example config for the backup cluster could look:
```yaml
type: postgis 
mode: standalone

cluster:
  useBarmanCloudPlugin: true
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters:
        barmanObjectName: my-cluster-object-store

  instances: 2
  env:
    - name: AWS_REQUEST_CHECKSUM_CALCULATION
      value: when_required
    - name: AWS_RESPONSE_CHECKSUM_VALIDATION
      value: when_required
  affinity:
    topologyKey: kubernetes.io/hostname

  monitoring:
    enabled: true

  roles:
    - name: app
      ensure: present
      comment: DB User for testing
      login: true
      superuser: false
      passwordSecret:
        name: app-user-credentials

backups:
  enabled: true
  endpointURL: "https://obs.eu-de.otc.t-systems.com" # If this is left out it hits AWS instead
  provider: s3
  secret:
    name: "my-cluster-backup-s3-creds"
  s3:
    region: "eu-de"
    bucket: "${vault:{{.Values.projectValues.context}}/data/{{.Values.projectValues.stage}}/storage/cloudnative_pg_bucket#bucket_name}"
    path: "/recovery/"
    # These still need to be set, even though these will be copied into the secret above (my-cluster-backup-s3-creds)
    # lest a templating error be thrown.
    accessKey: "${vault:{{.Values.projectValues.context}}/data/{{.Values.projectValues.stage}}/storage/cloudnative_pg_bucket#access_key}"
    secretKey: "${vault:{{.Values.projectValues.context}}/data/{{.Values.projectValues.stage}}/storage/cloudnative_pg_bucket#secret_key}"

  scheduledBackups:
    - name: test-backup
      schedule: "0 */5 * * * *"
      backupOwnerReference: self      
  retentionPolicy: "1d"

databases:
  - name: app             
    ensure: present       
    owner: app            
    template: template1   
    encoding: UTF8        
    schemas: []           
    extensions:           
      - name: postgis
      - name: postgis_topology
      - name: pg_prewarm
      - name: lo
      - name: fuzzystrmatch

```

The recovery cluster could look like this:
```yaml
type: postgis
mode: recovery

recovery:
  method: object_store
  endpointURL: "https://obs.eu-de.otc.t-systems.com"    # Hits AWS if omitted
  provider: s3
  clusterName: "my-cluster"                             # Must match the backup cluster's name
  secret:
    name: "my-recovery-cluster-backup-s3-creds"
  s3:
    region: "eu-de"
    bucket: "${vault:{{ .Values.projectValues.context }}/data/{{ .Values.projectValues.stage }}/storage/cloudnative_pg_bucket#bucket_name}"
    path: "/recovery/"
    accessKey: "${vault:{{ .Values.projectValues.context }}/data/{{ .Values.projectValues.stage }}/storage/cloudnative_pg_bucket#access_key}"
    secretKey: "${vault:{{ .Values.projectValues.context }}/data/{{ .Values.projectValues.stage }}/storage/cloudnative_pg_bucket#secret_key}"

cluster:
  useBarmanCloudPlugin: true
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      enabled: true
      isWALArchiver: true
      parameters:
        barmanObjectName: my-recovery-cluster-object-store

  instances: 2
  env:
    - name: AWS_REQUEST_CHECKSUM_CALCULATION
      value: when_required
    - name: AWS_RESPONSE_CHECKSUM_VALIDATION
      value: when_required
  affinity:
    topologyKey: kubernetes.io/hostname

  monitoring:
    enabled: true

  roles:
    - name: app
      ensure: present
      comment: DB User for testing
      login: true
      superuser: false
      passwordSecret:
        name: app-user-credentials

backups:
  enabled: false              # Could also be enabled like the example above
databases:
  - name: app                  
    ensure: present            
    owner: app                 
    template: template1        
    encoding: UTF8             
    schemas: []                
    extensions:                
      - name: postgis
      - name: postgis_topology
      - name: pg_prewarm
      - name: lo
      - name: fuzzystrmatch

```

For a full reference refer to the [comments in the values.yaml](https://github.com/Ninja243/cnpg-charts/blob/main/charts/cluster/values.yaml)
and for more examples, refer to [this folder from @pha91's repo](https://github.com/Ninja243/cnpg-charts/tree/main/charts/cluster/test/postgresql-cluster-configuration).