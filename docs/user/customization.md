# Cluster Customization

The OpenShift Installer allows for several different levels of customization. It is important to understand how and why each of these levels is exposed and the ramifications of making changes at each of these levels. This guide will attempt to walk through each of them and provide examples for why an administrator may want to make customizations.

Cluster customization can be broken into four major levels: OpenShift, Kubernetes, Platform, and OS. These four levels are rough abstraction layers (OpenShift being the highest layer and OS being the lowest) and fall into either the validated or unvalidated buckets. The levels within the validated bucket (OpenShift and Platform) encompass customization that is safe to perform - installation and automatic updates will succeed regardless of the changes made (to a reasonable degree). The levels within the unvalidated bucket (Kubernetes and OS) encompass customization that is not necessarily safe - after introducing changes, installation and automatic updates may not succeed.

## OpenShift Customization

The most simple customization is exposed by the installer as an interactive series of prompts. These prompts are required and represent a high-level of customization. They are needed in order to get a running OpenShift cluster, but they aren't enough to get anything other than a vanilla deployment out of the box. Further customization is possible once the cluster has been provisioned, but isn't covered in this document as it is a "Day 2" operation.

## Platform Customization

While the default cluster size may be sufficient for some, many will need to make alterations. This can include increasing the number of machines in the control plane, changing the type of the virtual machines that will be used (e.g. AWS instances), or adjusting the CIDR range used for the Kubernetes service network. This level of customization is exposed via the installer's `install-config.yaml`. The install-config can be accessed by running `openshift-install create install-config`. This file can then be modified as needed before running a later target.

The `install-config.yaml` generated by the installer will not have all of the available fields populated, so they will need to be manually added if they are needed. The full list of available fields can be found in the [Go Docs][godocs]. Documentation for each of the supported platforms can be found in their platform-specific section:

- [AWS][aws-customization]
- [Azure][azure-customization]

[aws-customization]: aws/customization.md
[azure-customization]: azure/customization.md
[godocs]: https://godoc.org/github.com/openshift/installer/pkg/types#InstallConfig

## Kubernetes Customization (unvalidated)

In addition to customizing OpenShift and aspects of the underlying platform, the installer allows arbitrary modification to the Kubernetes objects that are injected into the cluster. Note that there is currently no validation on the modifications that are made, so it is possible that the changes will result in a non-functioning cluster. The Kubernetes manifests can be viewed and modified using the `manifests` and `manifest-templates` targets.

The `manifests` target will render the manifest templates and output the result into the asset directory. Perhaps the most common use for this target is to include additional manifests in the initial installation. These manifests could be added after the installation as a "Day 2" operation, but there may be cases where they are necessary beforehand.

The `manifest-templates` target will output the unrendered manifest templates into the asset directory. This allows modification to the templates before they have been rendered, which may be useful to users who wish to reuse the templates between cluster deployments.

### Install Time Customization for Machine Configuration

**IMPORTANT**:

- These customizations require using the `manifests` target that does not provide compatibility guarantees, for more information [check here](versioning.md#versioning).
- This can affect upgradability of your cluster as the `machine-config-operator` can mark clusters tainted when user defined [MachineConfig][machine-config] objects are present in the cluster.

In most cases, user applications should be run on the cluster via Kubernetes workload objects (e.g. DaemonSet, Deployment, etc). For example, DaemonSets are the most stable way to run a logging agent on all hosts. However, there may be some cases where these workloads need to be executed prior to the node joining the Kubernetes cluster. For example, a compliance mandate like "the user must run auditing tools as soon as the operating system comes up" might require a custom systemd unit for an auditing container in the Ignition config for some or all nodes.

The configuration of machines in OpenShift is controlled using `MachineConfig` objects and what configuration is applied to a machine in the OpenShift cluster is based on the [MachineConfigPool][machine-config-pool] objects. To allow customization of machine configuration which is not possible as Day 2 operation, the installer allows users to bring their own custom `MachineConfig` objects.

1. `openshift-install --dir $INSTALL_DIR create manifests`

2. Copy files with `MachineConfig` objects to `$INSTALL_DIR/openshift/` directory.

    These custom `MachineConfig` objects are black boxes to the installer and the installer only plays the role of `oc create -f <custom-machine-config-object>` early enough into cluster bootstrap to make sure the configuration is used by the [MachineConfigOperator][machine-config-operator].

3. `openshift-install --dir $INSTALL_DIR create cluster`

#### Control plane with no Taints

All control plane nodes by default register with a taint `node-role.kubernetes.io/master=:NoSchedule` making them unschedulable by most normal workloads. An installation that requires the control plane to boot without that taint can push a custom `MachineConfig` object with a `kubelet.service` that doesn't include the taint.

For example:

1. Run `manifests` target to create all the manifests.

    ```console
    $ mkdir no-taint-cluster

    $ cp aws-install-config.yaml no-taint-cluster/install-config.yaml

    $ openshift-install --dir no-taint-cluster create manifests
    INFO Consuming "Install Config" from target directory

    $ ls -l no-taint-cluster/**
    no-taint-cluster/manifests:
    total 68
    -rw-r--r--. 1 xxxxx xxxxx  169 Feb 28 10:54 04-openshift-machine-config-operator.yaml
    -rw-r--r--. 1 xxxxx xxxxx 1589 Feb 28 10:54 cluster-config.yaml
    -rw-r--r--. 1 xxxxx xxxxx  149 Feb 28 10:54 cluster-dns-02-config.yml
    -rw-r--r--. 1 xxxxx xxxxx  243 Feb 28 10:54 cluster-infrastructure-02-config.yml
    -rw-r--r--. 1 xxxxx xxxxx  154 Feb 28 10:54 cluster-ingress-02-config.yml
    -rw-r--r--. 1 xxxxx xxxxx  557 Feb 28 10:54 cluster-network-01-crd.yml
    -rw-r--r--. 1 xxxxx xxxxx  327 Feb 28 10:54 cluster-network-02-config.yml
    -rw-r--r--. 1 xxxxx xxxxx  264 Feb 28 10:54 cvo-overrides.yaml
    -rw-r--r--. 1 xxxxx xxxxx  275 Feb 28 10:54 etcd-service.yaml
    -rw-r--r--. 1 xxxxx xxxxx  283 Feb 28 10:54 host-etcd-service-endpoints.yaml
    -rw-r--r--. 1 xxxxx xxxxx  268 Feb 28 10:54 host-etcd-service.yaml
    -rw-r--r--. 1 xxxxx xxxxx  118 Feb 28 10:54 kube-cloud-config.yaml
    -rw-r--r--. 1 xxxxx xxxxx 1299 Feb 28 10:54 kube-system-configmap-etcd-serving-ca.yaml
    -rw-r--r--. 1 xxxxx xxxxx 1304 Feb 28 10:54 kube-system-configmap-root-ca.yaml
    -rw-r--r--. 1 xxxxx xxxxx 3877 Feb 28 10:54 kube-system-secret-etcd-client.yaml
    -rw-r--r--. 1 xxxxx xxxxx 4030 Feb 28 10:54 machine-config-server-tls-secret.yaml
    -rw-r--r--. 1 xxxxx xxxxx  856 Feb 28 10:54 pull.json

    no-taint-cluster/openshift:
    total 28
    -rw-r--r--. 1 xxxxx xxxxx  293 Feb 28 10:54 99_binding-discovery.yaml
    -rw-r--r--. 1 xxxxx xxxxx  181 Feb 28 10:54 99_kubeadmin-password-secret.yaml
    -rw-r--r--. 1 xxxxx xxxxx  330 Feb 28 10:54 99_openshift-cluster-api_cluster.yaml
    -rw-r--r--. 1 xxxxx xxxxx 1015 Feb 28 10:54 99_openshift-cluster-api_master-machines-0.yaml
    -rw-r--r--. 1 xxxxx xxxxx 2655 Feb 28 10:54 99_openshift-cluster-api_master-user-data-secret.yaml
    -rw-r--r--. 1 xxxxx xxxxx 1750 Feb 28 10:54 99_openshift-cluster-api_worker-machineset.yaml
    -rw-r--r--. 1 xxxxx xxxxx 2655 Feb 28 10:54 99_openshift-cluster-api_worker-user-data-secret.yaml
    ```

2. Create a `MachineConfig` that includes `kubelet.service` that has no taints.

    ```sh
    cat > no-taint-cluster/openshift/99-master-kubelet-no-taint.yaml <<EOF
    apiVersion: machineconfiguration.openshift.io/v1
    kind: MachineConfig
    metadata:
      labels:
        machineconfiguration.openshift.io/role: master
      name: 02-master-kubelet
    spec:
      config:
        ignition:
          version: 2.2.0
        systemd:
          units:
          - contents: |
              [Unit]
              Description=Kubernetes Kubelet
              Wants=rpc-statd.service

              [Service]
              Type=notify
              ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests
              ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state
              EnvironmentFile=-/etc/kubernetes/kubelet-workaround
              EnvironmentFile=-/etc/kubernetes/kubelet-env

              ExecStart=/usr/bin/hyperkube \
                  kubelet \
                    --config=/etc/kubernetes/kubelet.conf \
                    --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig \
                    --rotate-certificates \
                    --kubeconfig=/var/lib/kubelet/kubeconfig \
                    --container-runtime=remote \
                    --container-runtime-endpoint=/var/run/crio/crio.sock \
                    --allow-privileged \
                    --node-labels=node-role.kubernetes.io/master \
                    --minimum-container-ttl-duration=6m0s \
                    --client-ca-file=/etc/kubernetes/ca.crt \
                    --cloud-provider=aws \
                    --volume-plugin-dir=/etc/kubernetes/kubelet-plugins/volume/exec \
                    \
                    --anonymous-auth=false \

              Restart=always
              RestartSec=10

              [Install]
              WantedBy=multi-user.target
            enabled: true
            name: kubelet.service
    EOF
    ```

    `machineconfiguration.openshift.io/role: master` label attaches this `MachineConfig` to the [master][master-machine-config-pool] `MachineConfigPool`. The [default][default-kubelet-service] configuration for the `kubelet.service` on libvirt includes the taint.

3. Run `cluster` target to create the cluster using the custom manifests.

    ```console
    $ openshift-install --dir no-taint-cluster create cluster
    INFO Consuming "Openshift Manifests" from target directory
    INFO Consuming "Master Machines" from target directory
    INFO Consuming "Common Manifests" from target directory
    INFO Creating infrastructure resources...
    INFO Waiting up to 30m0s for the Kubernetes API at https://api.test-cluster.example.com:6443...
    ...
    ```

    Check that no control plane nodes registered with taints:

    ```console
    $ oc --config no-taint-cluster/auth/kubeconfig get nodes -ojson | jq '.items[] | select(.metadata.labels."node-role.kubernetes.io/master" == "") | .spec.taints'
    null
    ```

    Check that the `02-master-kubelet` `MachineConfig` exists in the cluster:

    ```console
    oc --config no-taint-cluster/auth/kubeconfig get machineconfigs
    NAME                                                        GENERATEDBYCONTROLLER        IGNITIONVERSION   CREATED
    00-master                                                   3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    00-master-ssh                                               3.11.0-744-g5b05d9d3-dirty                     137m
    00-worker                                                   3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    00-worker-ssh                                               3.11.0-744-g5b05d9d3-dirty                     137m
    01-master-container-runtime                                 3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    01-master-kubelet                                           3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    02-master-kubelet                                                                        2.2.0             137m
    01-worker-container-runtime                                 3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    01-worker-kubelet                                           3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    99-master-3c81ffa3-3b8d-11e9-ac1e-52fdfc072182-registries   3.11.0-744-g5b05d9d3-dirty                     133m
    99-worker-3c83a226-3b8d-11e9-ac1e-52fdfc072182-registries   3.11.0-744-g5b05d9d3-dirty                     133m
    master-55491738d7cd1ad6c72891e77c35e024                     3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    worker-edab0895c59dba7a566f4b955d87d964                     3.11.0-744-g5b05d9d3-dirty   2.2.0             137m
    ```

[default-kubelet-service]: https://github.com/openshift/machine-config-operator/blob/master/templates/master/01-master-kubelet/_base/units/kubelet.yaml
[machine-config-operator]: https://github.com/openshift/machine-config-operator#machine-config-operator
[machine-config-pool]: https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfigController.md#machinepool
[machine-config]: https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfiguration.md
[master-machine-config-pool]: https://github.com/openshift/machine-config-operator/blob/master/manifests/master.machineconfigpool.yaml

## OS Customization (unvalidated)

In rare circumstances, certain modifications to the bootstrap and other machines may be necessary. The installer provides the "ignition-configs" target, which allows arbitrary modification to the [Ignition Configs][ignition] used to boot these machines. Note that there is currently no validation on the modifications that are made, so it is possible that the changes will result in a non-functioning cluster.

An example `worker.ign` is shown below. It has been modified to increase the HTTP timeouts used when fetching the generated worker config from the cluster. This isn't likely to be useful, but it does demonstrate what is possible.

```json ignition
{
  "ignition": {
    "version": "2.2.0",
    "config": {
      "append": [{
        "source": "https://test-cluster-api.example.com:22623/config/worker"
      }]
    },
    "security": {
      "tls": {
        "certificateAuthorities": [{
          "source": "data:text/plain;charset=utf-8;base64,LS0tLS1CRU..."
        }]
      }
    },
    "timeouts": {
      "httpResponseHeaders": 120
    }
  },
  "passwd": {
    "users": [{
      "name": "core",
      "sshAuthorizedKeys": [
        "ssh-ed25519 AAAA..."
      ]
    }]
  }
}
```

[ignition]: https://coreos.com/ignition/docs/latest/
