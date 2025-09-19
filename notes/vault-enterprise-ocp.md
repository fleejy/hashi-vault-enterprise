# Vault Enterprise on OpenShift

Reference Documents:
- https://developer.hashicorp.com/validated-designs/vault-operating-guides-adoption
- https://developer.hashicorp.com/validated-designs/vault-solution-design-guides-vault-enterprise
- https://developer.hashicorp.com/vault/docs/configuration
- https://github.com/hashicorp/vault-helm/blob/main/values.yaml
- [HashiCorp Docs: Run Vault on Openshift](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/helm/openshift)

## Vault Enterprise License
- license loaded via environment variable or configuration
- Loaded in the priority of 
  - `VAULT_LICENSE` > `VAULT_LICENSE_PATH` > `license_path`

## Vault configuration parameters
```yaml
# vault-helm/vaules.yaml
server:
  # If true, or "-" with global.enabled true, Vault server will be installed.
  # See vault.mode in _helpers.tpl for implementation details.
  enabled: "-"

  # [Enterprise Only] This value refers to a Kubernetes secret that you have
  # created that contains your enterprise license. If you are not using an
  # enterprise image or if you plan to introduce the license key via another
  # route, then leave secretName blank ("") or set it to null.
  # Requires Vault Enterprise 1.8 or later.
  enterpriseLicense:
    # The name of the Kubernetes secret that holds the enterprise license. The
    # secret must be in the same namespace that Vault is installed into.
    secretName: ""
    # The key within the Kubernetes secret that holds the enterprise license.
    secretKey: "license"

  image:
    repository: "hashicorp/vault"
    tag: "1.20.1"
    # Overrides the default Image Pull Policy
    pullPolicy: IfNotPresent

  # Configure the Update Strategy Type for the StatefulSet
  # See https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#update-strategies
  updateStrategyType: "OnDelete"

  # Ingress allows ingress services to be created to allow external access
  # from Kubernetes to access Vault pods.
  # If deployment is on OpenShift, the following block is ignored.
  # In order to expose the service, use the route section below
  ingress:
    enabled: false
    labels: {}
      # traffic: external
    annotations: {}

  # OpenShift only - create a route to expose the service
  # By default the created route will be of type passthrough
  route:
    enabled: false

    # When HA mode is enabled and K8s service registration is being used,
    # configure the route to point to the Vault active service.
    activeService: true

    labels: {}
    annotations: {}
    host: chart-example.local
    # tls will be passed directly to the route's TLS config, which
    # can be used to configure other termination methods that terminate
    # TLS at the router
    tls:
      termination: passthrough

  # Enables a headless service to be used by the Vault Statefulset
  service:
    enabled: true
    # Enable or disable the vault-active service, which selects Vault pods that
    # have labeled themselves as the cluster leader with `vault-active: "true"`.
    active:
      enabled: true
      # Extra annotations for the service definition. This can either be YAML or a
      # YAML-formatted multi-line templated string map of the annotations to apply
      # to the active service.
      annotations: {}
    # Enable or disable the vault-standby service, which selects Vault pods that
    # have labeled themselves as a cluster follower with `vault-active: "false"`.
    standby:
      enabled: true
      # Extra annotations for the service definition. This can either be YAML or a
      # YAML-formatted multi-line templated string map of the annotations to apply
      # to the standby service.
      annotations: {}

  # Run Vault in "HA" mode. There are no storage requirements unless the audit log
  # persistence is required.  In HA mode Vault will configure itself to use Consul
  # for its storage backend.  The default configuration provided will work the Consul
  # Helm project by default.  It is possible to manually configure Vault to use a
  # different HA backend.
  ha:
    enabled: false
    replicas: 3

    # Set the api_addr configuration for Vault HA
    # See https://developer.hashicorp.com/vault/docs/configuration#api_addr
    # If set to null, this will be set to the Pod IP Address
    apiAddr: null

    # Set the cluster_addr configuration for Vault HA
    # See https://developer.hashicorp.com/vault/docs/configuration#cluster_addr
    # If set to null, this will be set to https://$(HOSTNAME).{{ template "vault.fullname" . }}-internal:8201
    clusterAddr: null

    # Enables Vault's integrated Raft storage.  Unlike the typical HA modes where
    # Vault's persistence is external (such as Consul), enabling Raft mode will create
    # persistent volumes for Vault to store data according to the configuration under server.dataStorage.
    # The Vault cluster will coordinate leader elections and failovers internally.
    raft:

      # Enables Raft integrated storage
      enabled: false
      # Set the Node Raft ID to the name of the pod
      setNodeId: false

      # Note: Configuration files are stored in ConfigMaps so sensitive data
      # such as passwords should be either mounted through extraSecretEnvironmentVars
      # or through a Kube secret.  For more information see:
      # https://developer.hashicorp.com/vault/docs/platform/k8s/helm/run#protecting-sensitive-vault-configurations
      # Supported formats are HCL and JSON.
      config: |
        ui = true

        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          # Enable unauthenticated metrics access (necessary for Prometheus Operator)
          #telemetry {
          #  unauthenticated_metrics_access = "true"
          #}
        }

        storage "raft" {
          path = "/vault/data"
        }

        service_registration "kubernetes" {}

```
