# 🚀 Canary Deployment with Rollback: Upgrade Nginx v1 → v2 Using Ansible

This playbook upgrades Nginx from v1 to v2 using a canary deployment approach. If the upgrade fails on canary servers, it automatically rolls back to Nginx v1 to ensure service continuity.

## 📌 How It Works?
✅ **Step 1:** Upgrade Nginx v1 to v2 on canary servers (`web1`, `web2`)  
✅ **Step 2:** Validate the upgrade  
✅ **Step 3:** If successful, deploy to remaining servers (`web3`, `web4`, `web5`)  
✅ **Step 4:** If failed, rollback canary servers back to Nginx v1  

---

## 1️⃣ Inventory File (`inventory.ini`)

```ini
[canary]
web1
web2

[remaining]
web3
web4
web5

[all_servers:children]
canary
remaining
```

---

## 2️⃣ Playbook (`nginx_canary_upgrade.yml`)

```yaml
- name: Canary Deployment - Upgrade Nginx v1 to v2
  hosts: canary
  become: yes
  tasks:
    - name: Update package lists
      ansible.builtin.apt:
        update_cache: yes

    - name: Upgrade Nginx to Version 2 (Canary Servers)
      ansible.builtin.apt:
        name: nginx
        state: latest
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted

- name: Validate Nginx Upgrade on Canary Servers
  hosts: canary
  tasks:
    - name: Check Nginx Version
      ansible.builtin.command: nginx -v
      register: nginx_version
      ignore_errors: yes

    - name: Display Nginx Version
      ansible.builtin.debug:
        msg: "Nginx version installed: {{ nginx_version.stderr }}"

    - name: Rollback to Nginx v1 if upgrade fails
      block:
        - name: Fail if Nginx is not v2
          ansible.builtin.fail:
            msg: "Nginx upgrade failed on Canary servers!"
          when: "'nginx/2' not in nginx_version.stderr"

      rescue:
        - name: Rolling back to Nginx v1
          ansible.builtin.apt:
            name: nginx=1.18.0-0ubuntu1  # Specify the correct version for rollback
            state: present

        - name: Restart Nginx after rollback
          ansible.builtin.service:
            name: nginx
            state: restarted

- name: Deploy Nginx v2 to Remaining Servers if Canary Passed
  hosts: remaining
  become: yes
  tasks:
    - name: Update package lists
      ansible.builtin.apt:
        update_cache: yes

    - name: Upgrade Nginx to Version 2 (Remaining Servers)
      ansible.builtin.apt:
        name: nginx
        state: latest
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

---

## 3️⃣ Run the Playbook
Run the following command:

```sh
ansible-playbook -i inventory.ini nginx_canary_upgrade.yml
```

---

## 📌 What Happens If Upgrade Fails?
🚨 If Nginx v2 installation fails, Ansible:
✅ Detects failure using `nginx -v`  
✅ Triggers rollback to `nginx v1.18.0`  
✅ Restarts Nginx to restore service  

---

## 🔹 Best Practices
✔ **Use Version Pinning** (`nginx=1.18.0-0ubuntu1`) to control rollback versions  
✔ **Automate CI/CD Testing** before rolling out upgrades  
✔ **Use Blue-Green Deployment** for a safer upgrade approach  

---
