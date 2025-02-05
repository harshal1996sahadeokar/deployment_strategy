# 🚀 Canary Deployment Using Ansible

A Canary Deployment gradually shifts traffic to a new version of an application, reducing risk during deployments. Here’s a simple example using Ansible where we roll out a new version of a web app to a small set of servers first, then expand if everything is fine.

## 🛠 Scenario
We have 5 servers (`web1`, `web2`, `web3`, `web4`, `web5`). The new version (`v2`) of the app is first deployed to 2 canary servers (`web1` and `web2`). If the deployment is successful, we continue with the rest.

---

## 1️⃣ Inventory File (`inventory.ini`)
Define servers in groups (canary and remaining):

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

## 2️⃣ Playbook (`canary_deploy.yml`)

### Step 1: Deploy new version only to canary servers.
### Step 2: Wait & verify the application is running.
### Step 3: If successful, deploy to remaining servers.

```yaml
- name: Canary Deployment Using Ansible
  hosts: canary
  become: yes
  tasks:
    - name: Deploy new app version (v2) on Canary Servers
      ansible.builtin.copy:
        src: /path/to/app_v2
        dest: /var/www/html/index.html
      notify: Restart Web Server

  handlers:
    - name: Restart Web Server
      ansible.builtin.service:
        name: apache2
        state: restarted

- name: Validate Canary Servers
  hosts: canary
  tasks:
    - name: Check if the new app is running
      ansible.builtin.uri:
        url: "http://{{ inventory_hostname }}"
        return_content: yes
      register: response

    - name: Fail if application response is incorrect
      ansible.builtin.fail:
        msg: "Canary deployment failed!"
      when: "'New App Version v2' not in response.content"

- name: Deploy to Remaining Servers if Canary Passed
  hosts: remaining
  become: yes
  tasks:
    - name: Deploy new app version (v2) on Remaining Servers
      ansible.builtin.copy:
        src: /path/to/app_v2
        dest: /var/www/html/index.html
      notify: Restart Web Server

  handlers:
    - name: Restart Web Server
      ansible.builtin.service:
        name: apache2
        state: restarted
```

---

## 3️⃣ Execution Steps
Run the playbook:

```sh
ansible-playbook -i inventory.ini canary_deploy.yml
```

---

## 📝 What Happens?
✅ **Step 1:** Deploy new app version (`v2`) to canary servers (`web1`, `web2`).  
✅ **Step 2:** Check if the new version is running correctly.  
✅ **Step 3:** If successful, roll out to remaining servers (`web3`, `web4`, `web5`).  
✅ **Step 4:** If canary fails, stop further deployment!  

---

## 🔹 Advantages of This Approach
✔ **Risk Reduction** → If the update has issues, only `web1` and `web2` are affected.  
✔ **Quick Rollback** → If canary fails, you can restore the old version on just 2 servers.  
✔ **Automated Validation** → The playbook ensures the app is working before continuing deployment.  

---

🔹 **Contributors:** Feel free to modify and enhance this approach based on your use case!
