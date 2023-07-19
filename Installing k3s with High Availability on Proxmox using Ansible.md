# Installing k3s with High Availability on Proxmox using Ansible

This README provides instructions on how to install k3s, a lightweight Kubernetes distribution, with high availability (HA) on Proxmox using Ansible, a popular infrastructure automation tool. HA deployment ensures that your Kubernetes cluster remains highly available even in the event of node failures.

## Prerequisites

Before you begin, make sure you have the following prerequisites:

- Ansible installed on your control machine.
- Access to multiple Proxmox virtual machines (VMs) for deploying the Kubernetes cluster.

## Step 1: Setting up the Ansible Control Machine

1. Install Ansible on your control machine by following the official documentation: [Ansible Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html).
2. Create an Ansible inventory file (e.g., `inventory.ini`) that lists the VMs' details where you want to deploy the k3s cluster. Example content:
   ```ini
   [k3s_nodes]
   node1 ansible_host=<Proxmox_node_IP> ansible_user=<Proxmox_node_user> ansible_ssh_private_key_file=<path_to_private_key>
   node2 ansible_host=<Proxmox_node_IP> ansible_user=<Proxmox_node_user> ansible_ssh_private_key_file=<path_to_private_key>
   node3 ansible_host=<Proxmox_node_IP> ansible_user=<Proxmox_node_user> ansible_ssh_private_key_file=<path_to_private_key>
   ```
   Replace `<Proxmox_node_IP>`, `<Proxmox_node_user>`, and `<path_to_private_key>` with the actual details of your Proxmox VMs.
3. Ensure that you can SSH into the Proxmox VMs from your control machine without requiring a password. You can set up SSH key-based authentication or provide the necessary credentials within your Ansible inventory.

## Step 2: Creating an Ansible Playbook

1. Create a new directory for your Ansible playbook (e.g., `k3s-ha-playbook`).
2. Navigate to the newly created directory and create a new playbook file (e.g., `install_k3s.yml`).
3. Open the playbook file in a text editor and add the following content:

```yaml
---
- hosts: k3s_nodes
  become: true
  tasks:
    - name: Install dependencies
      apt:
        name: "{{ item }}"
        state: present
      with_items:
        - curl
        - jq

    - name: Install k3s master node
      command: curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--cluster-init --disable-agent" sh -

    - name: Retrieve master node token
      command: cat /var/lib/rancher/k3s/server/node-token
      register: master_token
      changed_when: false
      failed_when: master_token.stdout == ""

    - name: Join worker nodes to the cluster
      command: curl -sfL https://get.k3s.io | K3S_URL=https://{{ hostvars['node1']['ansible_host'] }}:6443 K3S_TOKEN={{ master_token.stdout }} sh -
      when: inventory_hostname != 'node1'
```

4. Save the playbook file.

## Step 3: Running the Ansible Playbook

1. Open a terminal and navigate to the directory where you created the playbook (`k3s-ha-playbook`).
2. Execute the following command to run the Ansible playbook and install k3s:

```bash
ansible-playbook -i inventory.ini install_k3s.yml
```

3. Ansible will connect to the Proxmox VMs and execute the playbook tasks, installing k3s on the master node (`node1`) and joining the worker nodes to the cluster.
4. Once the playbook execution completes successfully, you will have a k3s cluster with high availability running on your Proxmox VMs.

## Verifying the Cluster

To verify the cluster's status, you can SSH into the master node (`node1`) and use the `kubectl` command-line tool to interact with the cluster:

```bash
ssh <Proxmox_node_user>@<Proxmox_node_IP>
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces
```

You should see the master node and worker nodes listed as well as the running pods in the cluster.

## Conclusion

Congratulations! You have successfully installed a k3s cluster with high availability on Proxmox using Ansible. You can now start deploying and managing applications on your Kubernetes cluster. For more advanced configurations and options, refer to the official k3s and Ansible documentation.

**Note:** This README assumes a basic setup for a development or testing environment. For production deployments, consider additional security measures, such as securing your cluster communication, enabling RBAC, and configuring appropriate networking and storage options.