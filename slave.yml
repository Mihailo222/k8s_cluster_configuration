---
- hosts: slave_one
  become: true

  vars_files:
   - vars/vars.yml
   - vars/join.yml
  tasks:

   - name: Update master server.
     apt:
      update_cache: yes
      cache_valid_time: 3600

   - name: Install packages curl, ca-certificates, gnupg.
     apt:
      name: "{{ item.package }}"
      state: present
     with_items:
      - package: "curl"
      - package: "ca-certificates"
      - package: "gnupg"
   #################################### HOSTNAME CONFIGURATION ############################
   - name: Set hostname.
     command: hostnamectl set-hostname {{ slave_two_hostname }}

   - name: Add master host to /etc/hosts file.
     lineinfile:
      path: /etc/hosts
      line: "{{ item.node }}"
      state: present
     with_items:
      - node: "{{ master_ip }} {{ master_hostname }}"
      - node: "{{ slave_one_ip }} {{ slave_one_hostname }}"
      - node: "{{ slave_two_ip }} {{ slave_two_hostname }}"

   ######################################## ADD KERNEL MODULES ############################
   - name: Add overlay module and br_netfilter module as kernel modules to boot at load time.
     lineinfile:
      path: "/etc/modules-load.d/modules.conf"
      line: "{{ item.name }}"
      state: present
     with_items:
      - name: "overlay" #overlay is being used as storage driver in containers
      - name: "br_netfilter" #used for network filtering through Linux bridge in k8s cluster arch

   - name: Load newly added kernel modules in working memory without waiting for system restart.
     command: "sudo modprobe {{ item.module }}"
     with_items:
      - module: "overlay"
      - module: "br_netfilter"
  ######################################## ADD K8S NETWORKING MODULES ###########################

   - name: Create a file for k8s modules.
     file:
      path: "/etc/sysctl.d/99-kubernetes-cri.conf"
      state: touch

   - name: Add k8s modules to /etc/sysctl.d/99-kubernetes-cri.conf.
     lineinfile:
      path: /etc/sysctl.d/99-kubernetes-cri.conf
      line: "{{ item.module }}"
      state: present
     with_items:
      - module: "net.bridge.bridge-nf-call-iptables=1"
      - module: "net.ipv4.ip_forward=1"
      - module: "net.bridge.bridge-nf-call-ip6tables=1"
   ######################################## INSTALL CONTAINERD #################################
   - name: Install containerd on master node. (other docker packages are not needed)
     import_tasks: tasks/docker-on-ubuntu.yml

   - name: Output a default containerd configuration tp /etc/containerd/config.toml file.
     shell: |
      sudo containerd config default | sudo tee /etc/containerd/config.toml

   - name: Restart containerd service.
     service:
      name: containerd
      state: restarted
   ######################################## TURN SWAP OFF  #################################
   - name: Turn swap off.
     command: |
      sudo swapoff -a

   - name: Restart containerd daemon.
     service:
      name: containerd
      state: restarted
   ######################################## k8s INSTALLATION  #################################
   - name: Add k8s repository and install k8s packages.
     import_tasks: tasks/k8s_packages_on_ubuntu.yml

   ########################################### JOIN NODES  ################################

   - name: Join slave nodes to master k8s node (cluster).
     shell: "{{ join_command }}"
