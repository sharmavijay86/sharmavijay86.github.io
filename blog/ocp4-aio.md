# OCP 4.2/4.3 All-In-One (UPI mode)

This document assume reader is familiar with the OCP4x installation process.

## Before Deployment

- Setup the `install-config.yaml` to deploy a single master and no workers
    ```
    apiVersion: v1
    baseDomain: example.com
    compute:
    - hyperthreading: Enabled
      name: worker
      replicas: 0
    controlPlane:
      hyperthreading: Enabled
      name: master
      replicas: 1
    metadata:
     name: aio
    networking:
        clusterNetworks:
        - cidr: 10.128.0.0/14
            hostPrefix: 23
        networkType: OpenShiftSDN
        serviceNetwork:
        - 172.30.0.0/16
    platform:
        none: {}
    pullSecret: '<your-pull-secret-here>'
    sshKey: 'ssh-rsa AAA...'
    ```

## During Deployment
- During installation there still need for a temporary external load balancer (or poor man version, modify the DNS entries). 
    - For the installation prepare the DNS equivalent to this:
        ```
        aio.example.com             <ip_aio>
        etcd-0.aio.example.com      <ip_aio>
        apps.aio.example.com        <ip_aio>
        *.apps.aio.example.com      <ip_aio>
        api-int.aio.example.com     <ip_bootstrap>
        api.aio.example.com         <ip_bootstrap>

        # etcd Service Record
        _etcd-server-ssl._tcp.aio.example.com.   IN SRV  0   0   2380    etcd-0.aio.example.com.
        ```
    - After `bootkube.service` completes modify the DNS 
        ```
        aio.example.com             <ip_aio>
        etcd-0.aio.example.com      <ip_aio>
        apps.aio.example.com        <ip_aio>
        *.apps.aio.example.com      <ip_aio>
        api-int.aio.example.com     <ip_aio>
        api.aio.example.com         <ip_aio>

        # etcd Service Record
        _etcd-server-ssl._tcp.aio.example.com.   IN SRV  0   0   2380    etcd-0.aio.example.com.
        ```

- The single node will be shown with both roles (master and worker)
    ```
    $ oc get nodes
    NAME   STATUS   ROLES           AGE    VERSION
    aio    Ready    master,worker   33m    v1.16.2
    ```

- Set `etcd-quorum-guard` to unmanaged state 
    ```
    oc patch clusterversion/version --type='merge' -p "$(cat <<- EOF
    spec:
      overrides:
        - group: apps/v1
          kind: Deployment
          name: etcd-quorum-guard
          namespace: openshift-machine-config-operator
          unmanaged: true
    EOF
    )"
    ```
- Downscale `etcd-quorum-guard` to one:
    ```
    oc scale --replicas=1 deployment/etcd-quorum-guard -n openshift-machine-config-operator
    ```
- Downscale the number of routers to one:
    ```
    oc scale --replicas=1 ingresscontroller/default -n openshift-ingress-operator
    ```

- (Recommended) Downscale the number of consoles, authentication, OLM and monitoring services to one:
    ```
    oc scale --replicas=1 deployment.apps/console -n openshift-console
    oc scale --replicas=1 deployment.apps/downloads -n openshift-console

    oc scale --replicas=1 deployment.apps/oauth-openshift -n openshift-authentication

    oc scale --replicas=1 deployment.apps/packageserver -n openshift-operator-lifecycle-manager
    
    # NOTE: When enabled, the Operator will auto-scale this services back to original quantity
    oc scale --replicas=1 deployment.apps/prometheus-adapter -n openshift-monitoring
    oc scale --replicas=1 deployment.apps/thanos-querier -n openshift-monitoring
    oc scale --replicas=1 statefulset.apps/prometheus-k8s -n openshift-monitoring
    oc scale --replicas=1 statefulset.apps/alertmanager-main -n openshift-monitoring
    ```
- (optional) Setup image-registry to use ephemeral storage.

    **WARNING**: Only use ephemeral storage for internal registry for testing purposes.
    ```
    oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
    --patch '{"spec":{"storage":{"emptyDir":{}}}}'

    oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
    --patch '{"spec":{"managementState":"Managed"}}'
    ```
    NOTE: Wait until the `image-registry` operator completes the update before using the registry.
