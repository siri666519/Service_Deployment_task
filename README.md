# Service_Deployment_task




# Service_Deployment_task
Document: Service Deployment & Troubleshooting — Step-by-Step (Ready-to-copy)

Title: Service Deployment and Troubleshooting using Ansible, HAProxy, Nginx & Docker
Short: Deploy 3 Ubuntu servers (Ansible control, Load-balancer, Backend), set up HAProxy LB, Nginx web nodes, deploy Docker container on backend (Triton placeholder), test and document.

PREPARATION — summary of what you’ll create

EC2 instances (3):

Control (Ansible) — your workstation / VM where you run Ansible

VM1 — Load Balancer (HAProxy)

VM2 — Backend (Nginx + Docker/Triton)

Key pair (.pem)

Security groups for connectivity

Ansible inventory + playbook + templates

Tests and screenshots

A. Create EC2 instances (Console or CLI)
A.1 AWS Console (quick)

EC2 → Launch Instance → Ubuntu Server 22.04 LTS (x86_64).

Instance type:

Control: t3.small (optional) — you may use local machine

VM1 (lb): t3.medium (2 vCPU, 4GB)

VM2 (backend): t3.large (2 vCPU, 8GB) — if GPU needed use g4dn.xlarge

Add storage: 30 GB gp3

Security Group (create new sg-ansible-lb-backend):

For Control (if EC2): allow SSH from your IP

For VM1 (lb):

Inbound: SSH(22) from control IP; HTTP(80) 0.0.0.0/0; HTTP(8080) 0.0.0.0/0

Outbound: All

For VM2 (backend):

Inbound: SSH(22) from control IP; HTTP(80) from anywhere or from LB IP; port 8000 from LB IP

Outbound: All

Key pair: create/download devops-key.pem and keep safe.

Launch 3 instances (or 2 + control local).

Screenshot tips: capture EC2 dashboard showing the 3 instances with their public/private IPs & key pair name.

A.2 AWS CLI (optional — single command per instance)

Replace <AMI> with Ubuntu 22.04 AMI for your region, and <subnet-id> etc. Minimal example:

aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --count 1 \
  --instance-type t3.medium \
  --key-name devops-key \
  --security-group-ids sg-xxxx \
  --subnet-id subnet-xxxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=lb-server}]'


Repeat for backend (change instance-type and Name tag).

B. Prepare your control machine (Ansible server)

If using a separate EC2 as control or your laptop with Ubuntu:

B.1 Install Ansible & tools
sudo apt update -y
sudo apt install -y ansible sshpass git curl

B.2 Place your private key on control

Put devops-key.pem in ~/ (control).

Secure it:

chmod 400 ~/devops-key.pem

B.3 Create project folder
mkdir ~/ansible-project && cd ~/ansible-project
mkdir templates


Screenshot: terminal showing ansible --version and ls -l ~/devops-key.pem.

C. Inventory (hosts.ini)

Create hosts.ini in ~/ansible-project/:

[lb]
vm1 ansible_host=13.51.72.65 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/devops-key.pem

[backend]
vm2 ansible_host=13.60.233.138 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/devops-key.pem

[control]
localhost ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3


Replace IPs with your actual instance public/private IPs and key path.

Screenshot: cat hosts.ini output.

D. Test connectivity with Ansible
ansible -i hosts.ini all -m ping


You should see pong for vm1 and vm2.

Screenshot: the ansible -i hosts.ini all -m ping output.

Troubleshoot:

If Permission denied (publickey) → check key path, chmod 400, key pair used in EC2 instance.

E. Ansible playbook — site.yml (complete)

Create site.yml in ~/ansible-project/ with this content (copy exactly):

- hosts: all
  become: true
  tasks:
    - name: update apt and upgrade
      apt:
        update_cache: yes
        upgrade: dist

    - name: disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart ssh

    - name: install ufw
      apt:
        name: ufw
        state: present

    - name: allow ssh in ufw
      ufw:
        rule: allow
        port: 22
        proto: tcp

  handlers:
    - name: restart ssh
      service:
        name: ssh
        state: restarted

- hosts: all
  become: true
  tasks:
    - name: install nginx
      apt:
        name: nginx
        state: present

    - name: configure index.html with host identifier
      copy:
        dest: /var/www/html/index.html
        content: "Hello from {{ inventory_hostname }} ({{ ansible_host }})"
      notify: restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

- hosts: lb
  become: true
  tasks:
    - name: install haproxy
      apt:
        name: haproxy
        state: present

    - name: push haproxy config
      template:
        src: templates/haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
      notify: restart haproxy

  handlers:
    - name: restart haproxy
      service:
        name: haproxy
        state: restarted


Screenshot: cat site.yml.

F. HAProxy template — templates/haproxy.cfg.j2

Create templates/haproxy.cfg.j2 and paste:

global
    log /dev/log local0
    log /dev/log local1 notice
    daemon
    maxconn 2000

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend http_front
    bind *:80
    default_backend web_back

backend web_back
    balance roundrobin
    option httpchk GET /
    server vm1 127.0.0.1:80 check
    server vm2 {{ hostvars['vm2'].ansible_host }}:80 check

frontend triton_front
    bind *:8080
    default_backend triton_back

backend triton_back
    balance roundrobin
    option httpchk GET /
    server triton {{ hostvars['vm2'].ansible_host }}:8000 check


Notes:

We treat VM1 as local web server 127.0.0.1:80.

hostvars['vm2'].ansible_host is rendered by Ansible.

Screenshot: cat templates/haproxy.cfg.j2.

G. Run the playbook

From control node:

cd ~/ansible-project
ansible-playbook -i hosts.ini site.yml


Watch output — tasks ok/changed should show success.

Screenshot: terminal output of ansible-playbook run.

H. Verify services manually
H.1 On VM1 (LB)
ssh -i ~/devops-key.pem ubuntu@<VM1_IP>
sudo systemctl status haproxy
sudo ss -tlnp | grep :80
sudo ss -tlnp | grep :8080

H.2 On VM2 (Backend)
ssh -i ~/devops-key.pem ubuntu@<VM2_IP>
sudo systemctl status nginx
curl http://127.0.0.1/


Screenshot(s):

systemctl status haproxy showing active

curl http://<VM1_IP>/ showing responses

I. If HAProxy failed to start — quick debug steps

Check config syntax:

sudo haproxy -c -f /etc/haproxy/haproxy.cfg


Check journal:

sudo journalctl -xeu haproxy.service | tail -n 50


If port 80 in use:

sudo ss -tlnp | grep :80
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo systemctl restart haproxy


Ensure /run/haproxy exists if PID file issues:

sudo mkdir -p /run/haproxy
sudo chown haproxy:haproxy /run/haproxy


Screenshot: any error and then success status after fix.

J. Phase 2 — Docker (on VM2) & run Triton placeholder
J.1 Install Docker (Ansible ad-hoc)

From control:

ansible -i hosts.ini backend -m apt -a "name=docker.io state=present update_cache=yes" --become
ansible -i hosts.ini backend -m service -a "name=docker state=started enabled=yes" --become


Or manually on VM2:

ssh -i ~/devops-key.pem ubuntu@<VM2_IP>
sudo apt update -y
sudo apt install -y docker.io
sudo systemctl enable --now docker
docker --version

J.2 Run sample container (Triton placeholder)
sudo docker run -d --name triton_app -p 8000:80 nginx


Verify:

sudo docker ps
curl http://127.0.0.1:8000/


Screenshot: docker ps and curl output.

J.3 Ensure HAProxy routes to :8080

On LB:

sudo systemctl restart haproxy


From control:

curl http://<VM1_IP>:8080/


Expect “Welcome to nginx!”

Screenshot: curl to port 8080.

K. Test load-balancing (final)

From control:

for i in {1..10}; do curl -s http://<VM1_IP>/; echo; done


You should see alternating responses from vm1 and vm2.

Screenshot: terminal showing alternating responses.

L. Troubleshooting notes (include in doc)

SSH / key issues → check chmod 400 and inventory path

Inventory parsing errors → fix [group] names and key=value format

HAProxy failing to start → check port collisions (stop nginx on LB) and haproxy -c

Docker container not reachable → ensure -p 8000:80 mapping and UFW allows connection from LB

Add one small “command & fix” box per error in doc.

M. Screenshots you must take (so I can build the PDF)

Capture these as PNGs (prefer terminal & full window):

EC2 Instances list (IDs, names, IPs).

cat hosts.ini

ansible -i hosts.ini all -m ping (pong output)

cat site.yml

cat templates/haproxy.cfg.j2

ansible-playbook -i hosts.ini site.yml terminal output (success)

sudo systemctl status haproxy (active)

sudo haproxy -c -f /etc/haproxy/haproxy.cfg (valid)

curl http://<VM1_IP>/ (show alternating results)

docker ps and curl http://127.0.0.1:8000/ on VM2

If you want, follow this naming pattern:

01_EC2_instances.png

02_hosts_ini.png

...
This makes arranging images into the PDF easy.

N. Create final PDF (I will do)

When you upload the screenshots (all or partial), I will:

Compose the document with headings, commands, and screenshot placement.

Produce a PDF and provide a download link.

O. Quick Checklist (copy to top of doc)

 Instances created & IPs recorded

 devops-key.pem saved & chmod 400

 hosts.ini created & tested (ansible ping)

 site.yml created & executed (ansible-playbook)

 HAProxy config validated & running (haproxy -c)

 Docker installed on backend & container running

 HAProxy routes to container on :8080

 Screenshots taken for each milestone
