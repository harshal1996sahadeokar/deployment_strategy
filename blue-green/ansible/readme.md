# 🚀 Blue-Green Deployment for Nginx Upgrade (v1 → v2) Using Ansible

## 🔹 Objective
Upgrade Nginx from v1 to v2 using a Blue-Green Deployment strategy to ensure zero downtime.

## 🔹 How It Works?
1. **Blue (Current Production)** runs Nginx v1 on `web-blue` instances.
2. **Green (New Version)** deploys Nginx v2 on `web-green` instances.
3. **Traffic Switching**: If `web-green` (Nginx v2) is stable, update the load balancer to switch traffic from Blue → Green.
4. **Rollback**: If `web-green` fails, revert traffic back to `web-blue`.

---
## 1️⃣ Inventory File (`inventory.ini`)
```ini
[blue]
web-blue1
web-blue2

[green]
web-green1
web-green2

[all_servers:children]
blue
green
```

---
## 2️⃣ Playbook (`nginx_blue_green.yml`)
```yaml
- name: Deploy Nginx v2 on Green Environment
  hosts: green
  become: yes
  tasks:
    - name: Update package lists
      ansible.builtin.apt:
        update_cache: yes

    - name: Install Nginx v2 on Green Servers
      ansible.builtin.apt:
        name: nginx
        state: latest
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted

- name: Validate Nginx v2 on Green Environment
  hosts: green
  tasks:
    - name: Check Nginx Version
      ansible.builtin.command: nginx -v
      register: nginx_version
      ignore_errors: yes

    - name: Display Nginx Version
      ansible.builtin.debug:
        msg: "Nginx version installed: {{ nginx_version.stderr }}"

    - name: Fail if Nginx is not v2
      ansible.builtin.fail:
        msg: "Nginx upgrade failed on Green servers!"
      when: "'nginx/2' not in nginx_version.stderr"

- name: Switch Load Balancer Traffic to Green
  hosts: localhost
  tasks:
    - name: Update Load Balancer Target to Green Servers
      ansible.builtin.command: >
        aws elb register-instances-with-load-balancer --load-balancer-name my-lb
        --instances $(aws ec2 describe-instances --filters "Name=tag:role,Values=green"
        --query "Reservations[*].Instances[*].InstanceId" --output text)

    - name: Deregister Blue Servers from Load Balancer
      ansible.builtin.command: >
        aws elb deregister-instances-from-load-balancer --load-balancer-name my-lb
        --instances $(aws ec2 describe-instances --filters "Name=tag:role,Values=blue"
        --query "Reservations[*].Instances[*].InstanceId" --output text)

- name: Rollback to Blue Environment if Green Fails
  hosts: localhost
  tasks:
    - name: If Green Fails, Switch Back to Blue
      ansible.builtin.command: >
        aws elb register-instances-with-load-balancer --load-balancer-name my-lb
        --instances $(aws ec2 describe-instances --filters "Name=tag:role,Values=blue"
        --query "Reservations[*].Instances[*].InstanceId" --output text)
      when: "'nginx/2' not in nginx_version.stderr"
```

---
## 3️⃣ Run the Playbook
```sh
ansible-playbook -i inventory.ini nginx_blue_green.yml
```

---
## 📌 What Happens?
✅ Step 1: Install Nginx v2 on Green servers (`web-green1`, `web-green2`)
✅ Step 2: Validate the upgrade
✅ Step 3: If successful, switch the Load Balancer (ELB) traffic from `web-blue` → `web-green`
✅ Step 4: If failed, rollback the Load Balancer to `web-blue`

---
## 🔹 Best Practices
✔ Use **Auto Scaling Groups (ASG)** for automatic Blue-Green rollout
✔ Implement **Health Checks** before switching traffic to `web-green`
✔ Use **DNS Switching (Route 53)** instead of direct ELB registration for rollback flexibility

Would you like me to add **Route 53 DNS-based switching** for rollback instead of ELB? 🚀
