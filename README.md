# Installing Redis and Elasticsearch Using Ansible Playbook

This document outlines the process of installing and configuring **Redis** and **Elasticsearch** using **Ansible Playbooks**. The project involves creating reusable Ansible roles to automate the installation and configuration process for both technologies. Additionally, Elasticsearch is configured to run in two scenarios: 

1. **Multiple nodes on the same machine**.
2. **Multiple nodes on separate machines**.

---

## 1. Project Structure

The project is structured as follows:

```plaintext
.
├── ansible.cfg
├── deploy.yml
├── inventory.ini
├── README.md
├── roles
│   ├── elasticsearch
│   │   ├── defaults
│   │   │   └── main.yml
│   │   ├── files
│   │   ├── handlers
│   │   │   └── main.yml
│   │   ├── meta
│   │   │   └── main.yml
│   │   ├── README.md
│   │   ├── tasks
│   │   │   └── main.yml
│   │   ├── templates
│   │   ├── tests
│   │   │   ├── inventory
│   │   │   └── test.yml
│   │   └── vars
│   │       └── main.yml
│   └── redis
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── README.md
│       ├── tasks
│       │   └── main.yml
│       ├── templates
│       ├── tests
│       │   ├── inventory
│       │   └── test.yml
│       └── vars
│           └── main.yml
├── secret.yml
├── test_elasticsearch.yml
├── test_redis.yml
└── vault-password.txt
```

---

## 2. Installation Steps

### 2.1 Using Ansible Galaxy to Initialize Roles

Roles for **Redis** and **Elasticsearch** were initialized using the `ansible-galaxy init` command. This ensures a standard folder structure for roles, making them modular and reusable.

### 2.2 Installing Redis

Redis was installed following the official website’s instructions. The role structure and playbooks make it easy to replicate the process across different environments.

---

## 3. Elasticsearch Installation Scenarios

### 3.1 Multiple Nodes on the Same Machine

In this scenario, multiple Elasticsearch nodes are configured on a single host. Below are the key steps performed by the **Ansible tasks** in `roles/elasticsearch/tasks/main.yml`:

#### Code Snippets and Explanation

- **Download and Verify the Elasticsearch Package**:
  ```yaml
  - name: Download Elasticsearch package
    get_url:
      url: "{{ elasticsearch_deb_url }}"
      dest: "/tmp/{{ elasticsearch_deb_file }}"
  
  - name: Verify SHA512 checksum
    command: "shasum -a 512 -c /tmp/{{ elasticsearch_deb_file }}.sha512"
    register: sha512_check
    failed_when: "'OK' not in sha512_check.stdout"
  ```
  These tasks ensure the Elasticsearch `.deb` package is downloaded and verified using a checksum file to guarantee its integrity.

- **Install Elasticsearch**:
  ```yaml
  - name: Install Elasticsearch
    apt:
      deb: "/tmp/{{ elasticsearch_deb_file }}"
      state: present
  ```
  Installs the Elasticsearch package on the machine.

- **Configure Nodes**:
  ```yaml
  - name: Customize configuration for second node
    copy:
      dest: /etc/elasticsearch-node2/elasticsearch.yml
      content: |
        cluster.name: {{ cluster_name }}
        node.name: {{ node_name_2 }}
        discovery.seed_hosts: [{{ discovery_seed_hosts | join(', ') }}]
        network.host: {{ network_host_2 }}
        http.port: {{ http_port_2 }}
  ```
  This task customizes the `elasticsearch.yml` configuration for additional nodes. Variables like `cluster_name`, `node_name`, `network_host`, and `http_port` are used for flexibility.

- **Create Systemd Services**:
  ```yaml
  - name: Create systemd override for second node
    copy:
      content: |
        [Unit]
        Description=Elasticsearch (Node 2)
        ...
      dest: /etc/systemd/system/elasticsearch-node2.service
  ```
  A new systemd service file is created to manage the second Elasticsearch node independently.

---

### 3.2 Multiple Nodes on Separate Machines

In this setup, each node is deployed on a different host. The `roles/elasticsearch/tasks/main.yml` tasks dynamically configure the nodes based on the `inventory_hostname` variable.

#### Key Tasks

- **Dynamic Node Configuration**:
  ```yaml
  - name: Configure Elasticsearch for Node 1
    copy:
      content: |
        cluster.name: {{ elasticsearch_node1_config.cluster_name }}
        node.name: {{ elasticsearch_node1_config.node_name }}
        discovery.seed_hosts:
        {% for host in elasticsearch_node1_config.discovery_seed_hosts %}
          - {{ host }}
        {% endfor %}
        network.host: {{ elasticsearch_node1_config.network_host }}
        http.port: {{ elasticsearch_node1_config.http_port }}
        transport.tcp.port: {{ elasticsearch_node1_config.transport_tcp_port }}
      dest: /etc/elasticsearch/elasticsearch.yml
    when: inventory_hostname == "node1"
  ```
  This task ensures that the configuration is tailored for each node. Discovery hosts and initial master nodes are dynamically set using the `vars/main.yml` file.

---

## 4. Variables and Customization

### Variables for the Same Machine Setup

Defined in `roles/elasticsearch/vars/main.yml`:

```yaml
cluster_name: my-cluster
discovery_seed_hosts:
  - "127.0.0.1:9300"
  - "127.0.0.1:9301"
  - "127.0.0.1:9302"
# Node 1
node_name_1: node-1
network_host_1: 0.0.0.0
http_port_1: 9200
```

### Variables for the Separate Machines Setup

Defined in `roles/elasticsearch/vars/main.yml`:

```yaml
elasticsearch_node1_config:
  cluster_name: "cluster-node1"
  node_name: "node-1"
  discovery_seed_hosts: ["192.168.1.2", "192.168.1.3"]
  cluster_initial_master_nodes: ["node-1"]
  network_host: "192.168.1.1"
  http_port: 9200
  transport_tcp_port: 9300
```

---

## 5. Conclusion

The key takeaway is the importance of configuring the **variables file** (`vars/main.yml`) or directly editing the **elasticsearch.yml** file for each setup. This ensures nodes can discover and communicate with one another, whether running on the same or separate machines. 

Additional variables can always be added to further customize the configuration, such as enabling security settings or setting custom memory limits.
