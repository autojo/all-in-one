# Git Repo to demonstrate the combined use of the 3 roles ol-k3s-prep, ansible-role-k3s, install-dashboard-nfs-awx



## Example Webserver Content
```bash
k3s
├── helm
│   ├── awx-operator-2.11.0.tgz
│   └── nfs-subdir-external-provisioner-4.0.18.tgz
├── k3s-selinux-1.4-1.el9.noarch.rpm
└── v1.27.1+k3s1
    ├── k3s
    └── k3s-airgap-images-amd64.tar.gz
```

## Example Playbook main.yaml
```yaml
---

- name: Prepare k3s nodes and build a cluster with HA control plane
  hosts: k3s_cluster
  remote_user: '{{ cfgsvc_pb_remote_user | default("root") }}'
  become: true
  vars:
    webserver: 'http://webserver.my.domain:8080/k3s'

# vars for prepare role
    k3s_allow_os_upgrade: false
    k3s_selinux_policy: k3s-selinux-1.4-1.el9.noarch.rpm
    k3s_interface_name: enp0s3
    k3s_keepalived: true
    k3s_api_ip: 1.2.3.4
    k3s_release_version: v1.27.1+k3s1

# vars for k3s role
    k3s_state: restarted
    k3s_airgap: true
    k3s_config_file: /etc/rancher/k3s/config.yaml
    k3s_etcd_datastore: true
    k3s_server:
      tls-san: api.my.domain
      snapshotter: overlayfs
      selinux: true
      disable: local-storage
    k3s_become: true
    # k3s_registries:
    #   mirrors:
    #     docker.io:
    #       endpoint:
    #         - "https://registry.my.domain"
    #       rewrite:
    #         "^(.*)/(.*)": "docker.io/$1/$2"

  roles:
    - name: Oracle Linux Preparations
      role: ol-k3s-prep
      tags: [all, prep]
    - name: Instlalation of K3S
      role: ansible-role-k3s
      tags: [all, k3s]


- name: Install Helm charts
  hosts: localhost
  gather_facts: false
  connection: local
  vars:
# vars for helm charts
    k3s_nfs_server: nfs.my.domain
    k3s_nfs_path: /srv/nfs/k3s
    webserver: 'http://webserver.my.domain:8080/k3s'

    k3s_dashboard_helm_chart: kubernetes-dashboard-6.0.8.tgz
    k3s_nfs_provisioner_helm_chart: nfs-subdir-external-provisioner-4.0.18.tgz
    k3s_awx_operator_helm_chart: awx-operator-2.12.2.tgz

    state: present

    instance_name: awx-test
    awx_replicas: 3

    service_type: ClusterIP

    ingress_type: ingress
    ingress_hosts: awx-demo.my.domain
    ingress_path: /
    ingress_path_type: Prefix

    ipv6_disabled: true

    projects_persistence: true
    projects_storage_class: myStorageClass
    projects_storage_size: 8Gi

  roles:
    - name: Installation of Kubernetes Dashboard, NFS CSI and AWX Operator + AWX Instance
      role: install-dashboard-nfs-awx
      tags: [all, helm]

```


## Example inventory.yaml

```yaml
---
k3s_cluster:
  hosts:
    ol91.my.domain:
      k3s_control_node: true
    ol92.my.domain:
      k3s_control_node: true
    ol93.my.domain:
      k3s_control_node: true
```

## Your directory should look like this

```bash
all-in-one
├── README.md
├── inventory.yml
├── main.yaml
└── roles
    ├── ansible-role-k3s
    ├── install-dashboard-nfs-awx
    └── ol-k3s-prep
```