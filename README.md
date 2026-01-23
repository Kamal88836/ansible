# ansible


Got it 👍
Below is a complete, practical Ansible automation that validates all your prerequisites and prepares the system for Redfish-based OpenShift Agent installation.
This matches exactly what you listed (ILO, connectivity, internet, ansible, webserver, nmstate).

What this automation does

From the automation host (haproxy / bastion) it will:

✅ Verify ILO Redfish API is enabled & reachable

✅ Verify ILO IP reachability

✅ Verify Internet access

✅ Verify required tools:

Ansible

Apache (httpd/apache2)

HTTP/2

nmstate binary

✅ Start & enable webserver

✅ Expose /var/www/html for ISO / ignition hosting

Directory structure
redfish-automation/
├── inventory.ini
├── site.yml
├── group_vars/
│   └── all.yml
└── roles/
    └── prereq-check/
        ├── tasks/
        │   └── main.yml

inventory.ini
[automation_host]
haproxy ansible_host=192.168.250.10 ansible_user=root


(Change IP/user as per your environment)

group_vars/all.yml
ilo_user: admin
ilo_password: admin123

ilo_ips:
  - 192.168.250.55
  - 192.168.250.56

redfish_endpoint: "/redfish/v1/Systems/1"

web_root: /var/www/html

site.yml (Main playbook)
- name: Redfish + Agent Based Install Prerequisite Automation
  hosts: automation_host
  become: yes
  roles:
    - prereq-check

roles/prereq-check/tasks/main.yml
---
### 1. Check Internet Connectivity
- name: Check internet connectivity
  command: ping -c 2 8.8.8.8
  register: internet_check
  ignore_errors: yes

- name: Fail if internet is not reachable
  fail:
    msg: "Internet is NOT reachable from automation host"
  when: internet_check.rc != 0


### 2. Check ILO IP reachability
- name: Ping ILO IPs
  command: ping -c 2 {{ item }}
  loop: "{{ ilo_ips }}"
  register: ilo_ping
  ignore_errors: yes

- name: Fail if any ILO IP is unreachable
  fail:
    msg: "ILO IP {{ item.item }} is NOT reachable"
  when: item.rc != 0
  loop: "{{ ilo_ping.results }}"


### 3. Validate Redfish API
- name: Check Redfish API on ILO
  uri:
    url: "https://{{ item }}{{ redfish_endpoint }}"
    method: GET
    user: "{{ ilo_user }}"
    password: "{{ ilo_password }}"
    validate_certs: no
    return_content: yes
  register: redfish_check
  loop: "{{ ilo_ips }}"

- name: Fail if Redfish API is not responding
  fail:
    msg: "Redfish API not accessible on {{ item.item }}"
  when: item.status != 200
  loop: "{{ redfish_check.results }}"


### 4. Check Ansible Installation
- name: Verify Ansible installation
  command: ansible --version
  register: ansible_check
  changed_when: false


### 5. Install Apache + HTTP2
- name: Install Apache and required modules
  package:
    name:
      - httpd
      - mod_http2
    state: present
  when: ansible_facts['os_family'] == "RedHat"

- name: Install Apache (Ubuntu)
  package:
    name:
      - apache2
    state: present
  when: ansible_facts['os_family'] == "Debian"


### 6. Enable & Start Web Server
- name: Start and enable web server
  service:
    name: "{{ 'httpd' if ansible_facts['os_family'] == 'RedHat' else 'apache2' }}"
    state: started
    enabled: yes


### 7. Check Web Server Port
- name: Verify HTTP service
  uri:
    url: "http://localhost"
    status_code: 200


### 8. Check nmstate binary
- name: Check nmstate binary
  command: nmstatectl --version
  register: nmstate_check
  ignore_errors: yes

- name: Fail if nmstate is missing
  fail:
    msg: "nmstatectl binary NOT found"
  when: nmstate_check.rc != 0


### 9. Prepare ISO hosting directory
- name: Create ISO directory
  file:
    path: "{{ web_root }}/iso"
    state: directory
    mode: '0755'

- name: Show success message
  debug:
    msg: "All prerequisites validated successfully. System is READY for Redfish automation 🚀"

How to run
cd redfish-automation
ansible-playbook -i inventory.ini site.yml
