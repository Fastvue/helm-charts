# Fastvue Reporter Helm Chart

# Intro

Helm is a package manager for Kubernetes that provides a convenient way to deploy applications using pre-configured templates called charts. Fastvue Reporter can be easily deployed to a Kubernetes cluster using our official Helm charts, which handle all the necessary resources like pods, services, ingress rules, and persistent storage. This approach simplifies deployment, configuration management, and upgrades while ensuring consistent installations across different environments.

# Deploying

## Install Helm

Follow instructions at [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

## Add Helm Chart Repo

To deploy an instance, first add the Fastvue chart repo to Helm;

```bash
helm repo add fastvue https://fastvue.github.io/helm-charts
helm repo update
```

## Deploy Instance

Fastvue Reporter can be deployed with `helm install` , however there are some required values that must be specified. See [Required values](#required-values) for details.

We recommend specifying values via a separate values file;

**reporter-values.yml**

```yaml
namespace: reporter-test
hostname: reporter-test.example.com
recordPrefix: fvr-test
image:
  brand: fortigate
  tag: 1.0.2.91
```

```bash
helm install reporter-test fastvue/fastvue-reporter -f reporter-values.yml
```

The values can be specified in multiple files, so if you want to run multiple instances that have common values as well as some instance-specific values, you can create two files and deploy an instance by including both;

**reporter-common.yml**

```yaml
image:
  brand: fortigate
  tag: 1.0.2.91

elasticsearch:
  uri: https://es-cluster.es.region.azure.elastic-cloud.com/
  apiKey: REPLACE_WITH_API_KEY

auth:
  accessRequire: claim roles:fastvue-viewers
  adminRequire: claim roles:fastvue-admins
  oidc:
    cryptoPassphrase: change-me
    providerMetadataURL: https://oidc-provider-url
    clientID: client-id
    clientSecret: client-secret
```

**reporter-instance-abc123.yml**

```yaml
namespace: reporter-abc123
hostname: reporter-abc123.example.com
recordPrefix: fvr-abc123
```

**reporter-instance-xyz789.yml**

```yaml
namespace: reporter-xyz789
hostname: reporter-xyz789.example.com
recordPrefix: fvr-xyz789
```

Then instances `abc123` and `xyz789` can be deployed with these two commands;

```yaml
helm install reporter-abc123 fastvue/fastvue-reporter -f reporter-common.yml -f reporter-abc123.yml
helm install reporter-xyz789 fastvue/fastvue-reporter -f reporter-common.yml -f reporter-xyz789.yml
```

## Deploy Instance with Command Line Parameters

Alternatively, values can be specified either on the command line when running `helm install`;

```yaml
helm install reporter-test fastvue/fastvue-reporter --set namespace=reporter-test --set hostname=reporter-test.aks-dev.fastvue.co --set recordPrefix=fvr-test --set image.brand=fortigate --set image.tag=1.0.2.91
```

However to specify all needed values the command may become very long, so we recommend using a separate values file.

## Update Instance

To update an existing Fastvue Reporter deployment with new values or a new image version, first update your helm chart repository to ensure you have the latest version of the chart:

```bash
helm repo update
```

Then use `helm upgrade`:

```bash
helm upgrade reporter-abc123 fastvue/fastvue-reporter -f reporter-common.yml -f reporter-abc123.yml
```

You can also update specific values using the `--set` flag:

```bash
helm upgrade reporter-abc123 fastvue/fastvue-reporter --set image.tag=1.0.2.92
```

## Restarting the Service

After updating a deployment, you may need to restart the pod if the service doesn't start properly or updated configuration is not applied.

To ensure the Kubernetes pod has the latest configuration, you can restart the pod with the following command:

```bash
kubectl -n reporter-abc123 rollout restart deployment reporter
```

If the Fastvue Reporter service inside the pod is not running or unresponsive, you can restart it with the following command:

```bash
# Get the pod name
kubectl -n reporter-abc123 get pods

# Restart the Fastvue Reporter service
kubectl -n reporter-abc123 exec reporter-a1b2c3d4e5-xyz89 -- supervisorctl restart fastvue.reporter
```

## Check Status and Values

To check the status of your deployment:

```bash
helm status reporter-abc123
```

To see what values are currently set:

```bash
helm get values reporter-abc123
```

To list all installed deployments:

```bash
helm list
```

## Uninstall Instance

To completely remove a Fastvue Reporter deployment and all its associated resources:

```bash
helm uninstall reporter-abc123
```

**⚠️ WARNING: This will DELETE all data associated with the deployment, including:**
- The Kubernetes namespace
- Persistent volumes and their data
- All configuration and settings

If you want to preserve the data, you should backup the data first. See [Backup and Restore](#backup-and-restore) for details.

## Backup and Restore

### Backup Data

To backup your Fastvue Reporter data before uninstalling or for migration purposes:

```bash
# Get the pod name that's using the PVC
kubectl get pods -n reporter-abc123

# Create a backup (example for a pod named 'reporter-a1b2c3d4e5-xyz890')
kubectl cp reporter-abc123/reporter-a1b2c3d4e5-xyz89:/data/reporter ./backup-data
```

This will copy all Fastvue Reporter data including:
- Settings and configuration
- Alerts and reports
- Local data and cache files

### Restore Data

To restore data to a running Fastvue Reporter instance:

1. **Stop the Fastvue Reporter service:**
	```bash
	# Execute command in the running pod
	kubectl exec -n reporter-abc123 reporter-a1b2c3d4e5-xyz89 -- supervisorctl stop fastvue.reporter
	```

2. **Delete the existing data in the pod:**
	```bash
	# Delete the existing data
	kubectl exec -n reporter-abc123 reporter-a1b2c3d4e5-xyz89 -- rm -rf /data/reporter
	```

3. **Restore the backup data:**
	```bash
	# Copy your backup data to the pod
	kubectl cp ./backup-data reporter-abc123/reporter-a1b2c3d4e5-xyz89:/data/reporter
	```

4. **Set the correct ownership on the restored data:**
	```bash
	# Set the correct ownership on the restored data
	kubectl exec -n reporter-abc123 reporter-a1b2c3d4e5-xyz89 -- chown -R fastvue:fastvue /data/reporter
	```

5. **Start the Fastvue Reporter service:**
	```bash
	# Execute command in the running pod
	kubectl exec -n reporter-abc123 reporter-a1b2c3d4e5-xyz89 -- supervisorctl start fastvue.reporter
	```

**Important Notes:**
- Always backup before restoring to avoid data loss
- The Fastvue Reporter service must be stopped during restore to prevent data corruption
- Ensure the backup data structure matches the expected `/data/reporter` directory structure
- After restore, you may need to restart the pod if the service doesn't start properly. See [Restarting the Service](#restarting-the-service) for details.

# Values

## Required values

These values must be specified and must be unique when deploying an instance.

```yaml
namespace: fastvue-reporter-abc123
hostname: reporter-abc123.example.com
recordPrefix: fvr-abc123

image:
  brand: brand
  tag: latest
```

- `namespace` - The Kubernetes namespace that will be created for this instance of Fastvue Reporter.
- `hostname` - The hostname that will be used to access this instance of Fastvue Reporter.
- `recordPrefix` - The prefix given to indexes created in Elasticsearch by this instance of Fastvue Reporter.
- `image` - Details of the Fastvue Reporter image to deploy. See [Image](#image) for details.

## Image

These values determine which brand and version of Fastvue Reporter gets deployed.

```yaml
image:
  brand: brand
  tag: latest
  pullPolicy: Always          
```

- `image` - Details of the Fastvue Reporter image to use
    - `brand` (required) - The brand of Fastvue Reporter to install (e.g. `sonicwall` or `fortigate`). See [https://hub.docker.com/u/fastvue?search=reporter](https://hub.docker.com/u/fastvue?search=reporter) for available brands.
    - `tag` (required) - The docker image tag to install, should be a specific build number (e.g. `1.0.2.91`). See [https://hub.docker.com/u/fastvue?search=reporter](https://hub.docker.com/u/fastvue?search=reporter) for tags available in each brand.
    - `pullPolicy` - Policy for pulling container image on deployment, defaults to `Always`, can be set to `Always`, `IfNotPresent`, or `Never`.

## Auth

This section configures authentication on the Fastvue Reporter instance by applying the specified auth settings to the Apache configuration inside the Fastvue Reporter container.

If the `auth` section is omitted, no authentication will be configured and anyone that can connect to the instance has full access. If the `auth` section is included, then one of `basic`, `ldap`, or `oidc` must also be specified within the `auth` section.

**You must specify at least one of `accessRequire` or `authFile`, and at least one of `basic`, `ldap`, or `oidc`.**

```yaml
auth:
  accessRequire: claim roles:fastvue-viewers      # Grants read-only UI access
  adminRequire: claim roles:fastvue-admins        # Grants admin/API actions
```

Alternatively, you can directly provide the Apache auth directives via `authFile` and `authApiFile`:

```yaml
auth:
  authFile: |
    Require valid-user
  basic:
    authpasswd: |-
      admin:$apr1$randomsalt$hashedpassword
```

For separate admin/API access, you can also provide `authApiFile`:

```yaml
auth:
  authFile: |
    Require valid-user
  authApiFile: |
    Require user admin
  basic:
    authpasswd: |-
      admin:$apr1$randomsalt$hashedpassword
      viewer:$apr1$randomsalt$hashedpassword
```

The `accessRequire` and `adminRequire` are written in Apache’s `Require` syntax (see [https://httpd.apache.org/docs/current/mod/mod_authz_core.html#require](https://httpd.apache.org/docs/current/mod/mod_authz_core.html#require)).

Each auth mode has a different `Require` syntax, detailed below.

- Use one of two ways to specify viewer/admin access;
	- Using `accessRequire`/`adminRequire`
		- `accessRequire` - Specifies the `Require` directive that grants access to Fastvue Reporter. If `adminRequire` is also specified, then `accessRequire` determines read-only access. If `adminRequire` is NOT specified, then `accessRequire` determines full access.
		- `adminRequire` - Specifies the `Require` directive that grants access to call APIs that modify Fastvue Reporter’s state.
	- Using `authFile`/`authApiFile`
		- `authFile` - Provide the Apache auth directives for viewer access to Fastvue Reporter. If `authApiFile` is omitted, then `authFile` is used for both viewer and admin access.
		- `authApiFile` - Provide the Apache auth directives for admin access to Fastvue Reporter.
- `paths` - Optional set of directives to determine access to specific paths or APIs. See [Paths](#paths).

### Basic

Basic authentication using native browser login prompt (HTTP Authentication), authenticated by provided passwords list in `authpasswd` value.

```yaml
auth:
  accessRequire: valid-user
  adminRequire: user admin
  basic:
    authpasswd: |-
      admin:$apr1$randomsalt$hashedpassword
      viewer:$apr1$randomsalt$hashedpassword
```

The values for the `authpasswd` file can be generated using the `htpasswd -n (username)`  command;

```bash
# htpasswd -n admin
New password: 
Re-type new password: 
admin:$apr1$BAhmPQD3$gdInP98HjXjsEmR8Hujmu/

# htpasswd -n viewer
New password: 
Re-type new password: 
viewer:$apr1$M9fJ5dPA$wIE.Qg7ZlhfVgCfgzMsoV.
```

The Require syntax for Basic auth can be either `valid-user` for any user that can log in, or `user (username)` to require a specific user.

### LDAP

LDAP authentication using native browser login prompt (HTTP Authentication).

```yaml
auth:
  accessRequire: ldap-group cn=fastvue-viewers,ou=groups,dc=example,dc=com
  adminRequire: ldap-group cn=fastvue-admins,ou=groups,dc=example,dc=com
  ldap:
    authLDAPURL: ldap://ldap-server:389/dc=example,dc=com?uid?sub?(objectClass=*)
    bindDN: cn=admin,dc=example,dc=com
    bindPassword: password
    groupAttributeIsDN: true
    groupAttribute: member
    cert: |-
      -----BEGIN CERTIFICATE-----
      MIID...
      -----END CERTIFICATE-----
    verifyCert: true
```

- `authLDAPURL` - LDAP authentication URL
- `bindDN` - LDAP bind DN
- `bindPassword` - LDAP bind password
- `groupAttributeIsDN` - LDAP group attribute is DN (boolean, default: true)
- `groupAttribute` - LDAP group attribute (default: member)
- `cert` - (string, optional) PEM-encoded certificate to use for LDAP server validation.
- `verifyCert` - (boolean, optional) Whether to verify the LDAP server certificate.

The Require syntax usable in LDAP auth is documented at https://httpd.apache.org/docs/2.4/mod/mod_authnz_ldap.html#requiredirectives. Note that when using the `Require ldap-group` directive, the DN must NOT be wrapped in quotes as this will cause the match to fail.

### OIDC

OIDC authentication using an external auth provider.

```yaml
auth:
  accessRequire: claim roles:fastvue-viewers
  adminRequire: claim roles:fastvue-admins
  oidc:
    cryptoPassphrase: change-me                    # Any random string ≥32 chars
    providerMetadataURL: https://oidc-provider-url # Well-known metadata URL
    clientID: client-id                            # Client / application ID
    clientSecret: client-secret                    # Client secret
```

OIDC auth in Apache uses mod_auth_openidc ([https://github.com/OpenIDC/mod_auth_openidc](https://github.com/OpenIDC/mod_auth_openidc)), and their Require syntax is documented at [https://github.com/OpenIDC/mod_auth_openidc/wiki/Authorization](https://github.com/OpenIDC/mod_auth_openidc/wiki/Authorization). The exact form of the Require syntax depends on which OIDC provider is in use, and how it is configured.

The value for `cryptoPassphrase` is a random string used for encrypting session state, and a value for it can be generated with `openssl rand -base64 32`.

### Paths

The `paths` section allows you to define access requirements for specific pages or APIs.

```yaml
auth:
  accessRequire: valid-user
  adminRequire: user admin
  paths: |
    Use PathRequire "/Settings/Productivity.aspx" "user admin"
    Use APIRequire "Settings\.Keywords\.|Alerts\." "valid-user"
    <Location "/Settings/Productivity.aspx">
      Require claim roles:fastvue-admins
    </Location>
```

Apache directives such as `<Location>` can be written in the `paths` section, but there are also some macros defined to allow convenient definitions of directives for a specific path or API. Macros must be invoked with the `Use` directive (e.g. `Use PathRequire ...`).

- `Use PathRequire (path) (require)` 

	Applies the specified `Require` directive to the specified path, for example;

	```
	Use PathRequire “/Settings/Productivity.aspx” “claim groups:fastvue-admins”`
	```
	
	Would restrict the Productivity settings page to Admins only.
- `Use APIRequire (api-spec) (require)` 
	
	Applies the specified `Require` directive to APIs matching the specified `api-spec`, which is a Regex defining the starting part of the API, e.g. `Settings\.Keywords\.` or `Alerts\.`. Note that all dots `.` must be escaped as `\.` to work in Regex. Multiple APIs can be combined with a pipe `|` symbol, e.g. `Settings\.Keywords\.|Alerts\.`, for example;
	
	```
	Use APIRequire “Settings\.Keywords\.|Alerts\.” “claim groups:fastvue-viewers”
	```
	
	Would allow the `fastvue-viewers` group to modify Keywords and dismiss Alerts even if they are restricted from doing so by the `authapi.conf` configuration.

## Elasticsearch

Configures a connection to an external Elasticsearch cluster. If omitted, Fastvue Reporter will use an internal Elasticsearch instance within its container.

```yaml
elasticsearch:
  uri: https://es-cluster.es.region.azure.elastic-cloud.com/
  apiKey: REPLACE_WITH_API_KEY
```

- `uri` - URI of the endpoint to connect to the Elasticsearch cluster on.
	
	Supported versions:

	- Elasticsearch 7.x
	- Elasticsearch 8.x
	- OpenSearch 2.x

- `apiKey` - API key used to authenticate to Elasticsearch (via an `Authorization: ApiKey (key)` header in requests).

## Resources

Configures the resources allocated to Fastvue Reporter.

```yaml
resources:
  cpu: "4"
  cpuMinimum: "0.25"
  memory: 4Gi
  storage: 5Gi
  storageClass: default
```

- `cpu` (optional, default: `"4"`) - Maximum CPU cores allowed for Fastvue Reporter (limits). Must be specified as a string with double-quotes.
- `cpuMinimum` (optional, default: `"0.25"`) - Minimum CPU cores reserved for Fastvue Reporter (requests). Must be specified as a string with double-quotes.
- `memory` (optional, default: `1Gi`) - The amount of memory reserved for Fastvue Reporter (requests).
- `storage` (optional, default: `5Gi`) - The size of the disk storage for Fastvue Reporter’s local state (Settings, Alerts, Reports, etc.)
- `storageClass` - (optional) Specify which Kubernetes storage class to use for the PVC. If not specified, the cluster's default storage class will be used.

## Syslog

Syslog configuration.

```yaml
syslog:
  port: 50514
```

- `port` - Port number to create a Kubernetes Service for. This will generally allocate a random IP address from an available pool on the platform Kubernetes is running on listening on this port to connect to the Syslog ingest port on the Fastvue Reporter container.

## Ingress

Configure the Ingress in Kubernetes for accessing the Fastvue Reporter interface. Omitting this section uses defaults, which usually uses the default certificate on the installed ingress controller.

```yaml
ingress:
  tls:
    azureKeyVaultURI: https://keyvault-name.vault.azure.net/certificates/cert-name
    secretName: keyvault-reporter-ingress
```

- `ingress.tls.azureKeyVaultURI` (Azure Only) - Specify the Azure KeyVault Certificate URL to use for TLS for this instance of Fastvue Reporter.
- `secretName` - Name of the TLS secret to use for the ingress.