# X-K8S SRIOV Manual 

## Pre-Configure Before you install X-K8S  

1. Make sure your SRIOV NIC PF is pluged in with cable and Linked-up.  
    You can check your SRIOV PF link status by  

    ```bash=
    ifconfig enp129s0f0 up
    ethtool enp129s0f0
    ```

2. Enable VF for your SRIOV NIC on each node.  

    e.g.  

    ```bash=
    echo 8 > /sys/class/net/enp129s0f0/device/sriov_numvfs
    ```

3. Edit/Create `/etc/pcidp/config.json` on node with SRIOV nic.
    e.g.

    ```json
    {
        "resourceList": [{
                "resourceName": "intel_sriov_netdevice",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["i40evf", "ixgbevf"]
                }
            },
            {
                "resourceName": "intel_sriov_dpdk",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["vfio-pci"],
                    "pfNames": ["enp0s0f0","enp2s2f1"]
                }
            },
            {
                "resourceName": "mlnx_sriov_rdma",
                "isRdma": true,
                "selectors": {
                    "vendors": ["15b3"],
                    "devices": ["1018"],
                    "drivers": ["mlx5_ib"]
                }
            },
            {
                "resourceName": "infiniband_rdma_netdevs",
                "isRdma": true,
                "selectors": {
                    "linkTypes": ["infiniband"]
                }
            }
        ]
    }
    ```

    `"resourceList"` should contain a list of config objects. Each config object may consist of following fields:

    |     Field      | Required |        Description        |                      Type - Accepted values                       |                                      Example/Accepted values                                       |
    |----------------|----------|---------------------------|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
    | "resourceName" | Yes      | Endpoint resource name    | string - must be unique and should not contain special characters | "sriov_net_A"                                                                                      |
    | "selectors"    | No       | A map of device selectors | Each selector is a map of string list.                            | "vendors": ["8086"], "devices": ["154c", "10ed"], "drivers": ["vfio-pci"], "pfNames": ["enp2s2f0"], "linkTypes": ["ether"] |
    | "isRdma"       | No       | Mount RDMA resources      | `bool` - boolean value true or false                              | "isRdma": true                                                                                     |

    You can check the PF bus address by `lshw -class network -businfo`  
    You can check the NIC's vendor and device ID by `lspci -nn`

## Install X-K8S

Before Install X-K8S, Check again following items are performed on each node you desire to use SRIOV.  

- Latest SRIOV NIC driver are installed.  
- SRIOV PF's status are all linked-up.  
- SRIOV VF are all created.
- `/etc/pcidp/config.json` are created and well configured.

## Start Installing X-K8S with SRIOV Support  

1. Edit `extraVars.yml` to enable sriov support.  

   ```yaml=
   ## SRIOV Support
   sriov_enabled : true
   ```

2. Install X-K8S

    ```bash=
    ./x-k8s install
    ```

## Validation  

When installation completes. You use following commands to check if SRIOV-support is successfully enabled.

```bash=
# Should see sriov-device-plugin daemonset is running.
root@node1:~# kubectl get ds -n kube-system
NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                   AGE
kube-flannel                     2         2         2       2            2           beta.kubernetes.io/os=linux     16h
kube-multus-ds-amd64             2         2         2       2            2           beta.kubernetes.io/arch=amd64   16h
kube-proxy                       2         2         2       2            2           beta.kubernetes.io/os=linux     16h
kube-sriov-device-plugin-amd64   2         2         1       2            1           beta.kubernetes.io/arch=amd64   16h
```

And you can check resouce for each node  

```bash=
# You should see sriov with the resouce name you defined in config.json
root@node1:~# kubectl get node node2 -o json | jq .status.capacity
{
  "cmk.intel.com/exclusive-cores": "10",
  "cpu": "36",
  "ephemeral-storage": "164140444Ki",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "intel.com/sriov_net_A": "8",
  "intel.com/sriov_net_B": "8",
  "memory": "65925044Ki",
  "pods": "110"
}

```

## Create Pod with SRIOV VF NIC

1. Create and apply custom network definitation that uses sriov nic from sriov pool.
    e.g.  

    ```yaml=
    apiVersion: "k8s.cni.cncf.io/v1"
    kind: NetworkAttachmentDefinition
    metadata:
      name: sriov-net-a
      annotations:
        k8s.v1.cni.cncf.io/resourceName: intel.com/sriov_net_A
    spec:
      config: '{
      "type": "sriov",
      "cniVersion" : "0.3.1",
      "vlan": 1000,
      "ipam": {
        "type": "host-local",
        "subnet": "10.56.217.0/24",
        "rangeStart": "10.56.217.171",
        "rangeEnd": "10.56.217.181",
        "routes": [{
          "dst": "0.0.0.0/0"
        }],
        "gateway": "10.56.217.1"
      }
    }'
    ```
    If you want to define the interface name go check [HERE](https://github.com/intel/multus-cni/blob/master/doc/how-to-use.md#lauch-pod-with-text-annotation-for-networkattachmentdefinition-in-different-namespace)

1. Create and apply the pod.
    e.g.

    ```yaml=
    apiVersion: v1
    kind: Pod
    metadata:
      name: testpod1
      annotations:
        k8s.v1.cni.cncf.io/networks: sriov-net-a
    spec:
      nodeSelector:
        kubernetes.io/hostname: node2
      containers:
      - name: appcntr1
        image: ubuntu
        imagePullPolicy: IfNotPresent
        command: [ "/bin/bash", "-c", "--" ]
        args: [ "while true; do sleep 300000; done;" ]
        resources:
          requests:
            intel.com/sriov_net_A: '1'
          limits:
            intel.com/sriov_net_A: '1'
    ```

2. You should see a nic in pod with the same net CIDR you define in "sriov-net-a"
