# vSphere cluster template parameters

## ClusterDeployment parameters

To create a cluster deployment a number of parameters should be passed to the
`ClusterDeployment` object.

### Parameter list

The following is the list of vSphere specific parameters that are _required_
for successful cluster creation.

| Parameter                             | Example                               | Description                                                     |
|---------------------------------------|---------------------------------------|-----------------------------------------------------------------|
| `.spec.config.vsphere.server`         | `vcenter.example.com`                 | Address of the vSphere instance                                 |
| `.spec.config.vsphere.thumbprint`     | `"00:00:00:..."`                      | Certificate thumbprint                                          |
| `.spec.config.vsphere.datacenter`     | `DC`                                  | Datacenter name                                                 |
| `.spec.config.vsphere.datastore`      | `/DC/datastore/DS`                    | Datastore path                                                  |
| `.spec.config.vsphere.resourcePool`   | `/DC/host/vCluster/Resources/ResPool` | Resource pool path                                              |
| `.spec.config.vsphere.folder`         | `/DC/vm/example`                      | Folder path                                                     |
| `.spec.config.controlPlane.network`   | `/DC/network/vm_net`                  | Network path for `controlPlane`                                 |
| `.spec.config.worker.network`         | `/DC/network/vm_net`                  | Network path for `worker`                                       |
| `.spec.config.*.ssh.publicKey`        | `"ssh-ed25519 AAAA..."`               | SSH public key in `authorized_keys` format                      |
| `.spec.config.*.vmTemplate`           | `/DC/vm/templates/ubuntu`             | VM template image path                                          |
| `.spec.config.controlPlaneEndpointIP` | `172.16.0.10`                         | `kube-vip` vIP which will be created for control plane endpoint |

To get the vSphere certificate thumbprint you can use the following command:

```bash
curl -sw %{certs} https://vcenter.example.com | openssl x509 -sha256 -fingerprint -noout | awk -F '=' '{print $2}'
```

[`govc`](https://github.com/vmware/govmomi/blob/main/govc/README.md), a vSphere CLI, can also help to discover proper values for some of the parameters:

```bash
# vsphere.datacenter
govc ls

# vsphere.datastore
govc ls /*/datastore/*

# vsphere.resourcePool
govc ls /*/host/*/Resources/*

# vsphere.folder
govc ls -l /*/vm/**

# controlPlane.network, worker.network
govc ls /*/network/*

# *.vmTemplate
govc vm.info -t '*'
```

> NOTE:
> Follow official `govc` installation instructions from [here](https://github.com/vmware/govmomi/blob/main/govc/README.md#installation). The `govc` usage guide is [here](https://github.com/vmware/govmomi/blob/main/govc/README.md#usage).
>
> Minimal `govc` configuration requires setting: `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD` environment variables.

### Configuring some of k0s, k0smotron parameters

* `k0s.arch` (string): Defines K0s Arch in its download URL. Available if [global.k0sURL](../../appendix/appendix-extend-mgmt.md#configuring-a-global-k0s-url)
   is set. Possible values: `"amd64"` (default), `"arm64"`, `"arm"`.
* `k0s.cpArgs` <sup>only standalone</sup> (array of strings): A list of extra arguments to be passed to k0s controller.
   See: <https://docs.k0sproject.io/stable/cli/k0s_controller>.
* `k0s.workerArgs` (array of strings): A list of extra arguments for configuring the k0s worker node. See: <https://docs.k0sproject.io/stable/cli/k0s_worker>.
* `k0smotron.controllerPlaneFlags` <sup>only hosted</sup> (array of strings): The `controllerPlaneFlags` parameter enables you to configure additional flags for the k0s control plane
   and to override existing flags. The default flags are kept unless they are explicitly overriden. Flags with arguments must be specified as a single
   string, such as `--some-flag=argument`.

## Example of a ClusterDeployment CR

With all above parameters provided your `ClusterDeployment` can look like this:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: cluster-1
spec:
  template: vsphere-standalone-cp-{{{ extra.docsVersionInfo.providerVersions.dashVersions.vsphereStandaloneCpCluster }}}
  credential: vsphere-credential
  config:
    clusterLabels: {}
    vsphere:
      server: vcenter.example.com
      thumbprint: "00:00:00"
      datacenter: "DC"
      datastore: "/DC/datastore/DC"
      resourcePool: "/DC/host/vCluster/Resources/ResPool"
      folder: "/DC/vm/example"
    controlPlaneEndpointIP: "<VSPHERE_SERVER>"
    controlPlane:
      ssh:
        user: ubuntu
        publicKey: |
          ssh-rsa AAA...
      rootVolumeSize: 50
      cpus: 2
      memory: 4096
      vmTemplate: "/DC/vm/template"
      network: "/DC/network/Net"
    worker:
      ssh:
        user: ubuntu
        publicKey: |
          ssh-rsa AAA...
      rootVolumeSize: 50
      cpus: 2
      memory: 4096
      vmTemplate: "/DC/vm/template"
      network: "/DC/network/Net"
```

Don't forget to replace placeholder values such as `VSPHERE_SERVER` with actual values for your environment.

## SSH

Currently SSH configuration on vSphere expects that the user is already created
before template creation. Because of that you must pass the username along with the SSH
public key to configure SSH access.

The SSH public key can be passed to `.spec.config.ssh.publicKey` (in the case of a
hosted control plane) or `.spec.config.controlPlane.ssh.publicKey` and
`.spec.config.worker.ssh.publicKey` (in the case of a standalone control) of the
`ClusterDeployment` object.

The SSH public key must be passed literally as a string.

You can pass the username to `.spec.config.controlPlane.ssh.user`,
`.spec.config.worker.ssh.user` or `.spec.config.ssh.user`, depending on you
deployment model.

## VM resources

The following parameters are used to define VM resources:

| Parameter         | Example | Description                                                              |
|-------------------|---------|--------------------------------------------------------------------------|
| `.rootVolumeSize` | `50`    | Root volume size in GB (can't be less than the one defined in the image) |
| `.cpus`           | `2`     | Number of CPUs                                                           |
| `.memory`         | `4096`  | Memory size in MB                                                        |

The resource parameters are the same for hosted and standalone CP deployments,
but they are positioned differently in the spec, which means that they're going to:

* `.spec.config` in case of hosted CP deployment.
* `.spec.config.controlPlane` in in case of standalone CP for control plane nodes.
* `.spec.config.worker` in in case of standalone CP for worker nodes.

## VM Image and network

To provide image template path and network path the following parameters must be
used:

| Parameter     | Example           | Description         |
|---------------|-------------------|---------------------|
| `.vmTemplate` | `/DC/vm/template` | Image template path |
| `.network`    | `/DC/network/Net` | Network path        |

As with resource parameters the position of these parameters in the
`ClusterDeployment` depends on deployment type and these parameters are used in:

* `.spec.config` in case of hosted CP deployment.
* `.spec.config.controlPlane` in in case of standalone CP for control plane nodes.
* `.spec.config.worker` in in case of standalone CP for worker nodes.
