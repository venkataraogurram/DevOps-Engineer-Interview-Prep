# Ansible cheatsheet

[← Cheatsheet index](README.md)

Ansible = agentless configuration management over SSH (or WinRM / SSM); YAML playbooks of idempotent modules. In an immutable-image world it runs inside the Packer build; in a mutable world it configures running fleets.

## CLI

```bash
ansible --version; ansible-config dump --only-changed; ansible-config init --disabled > ansible.cfg
ansible all -i inventory.ini -m ping; ansible web -m shell -a 'uptime' -b; ansible db -m setup -a 'filter=ansible_distribution*'
ansible all -m copy -a 'src=./f dest=/tmp/f mode=0644' -b; ansible all -m service -a 'name=nginx state=restarted' -b
ansible-inventory -i inventory --list; ansible-inventory --graph; ansible-inventory --host web1
ansible-playbook site.yml -i inventories/prod --limit web --tags nginx --skip-tags slow --check --diff
ansible-playbook site.yml -e 'env=prod' -e @vars.json --start-at-task 'Install nginx' --step -vvv
ansible-playbook site.yml --list-tasks --list-hosts --syntax-check
ansible-vault create secrets.yml; ansible-vault edit|view|encrypt|decrypt|rekey; ansible-vault encrypt_string 'pw' --name db_pass
ansible-playbook site.yml --ask-vault-pass | --vault-password-file ~/.vault_pass | --vault-id prod@prompt
ansible-galaxy init roles/nginx; ansible-galaxy install -r requirements.yml; ansible-galaxy collection install amazon.aws community.general
ansible-doc copy; ansible-doc -l | grep aws; ansible-doc -t lookup aws_ssm
ansible-lint site.yml; molecule test
ansible-pull -U git@repo site.yml      # pull mode from cron
```

## Inventory

```ini
# inventory.ini
[web]
web1 ansible_host=10.0.1.10
web2 ansible_host=10.0.1.11
[db]
db1 ansible_host=10.0.2.10 ansible_user=admin
[prod:children]
web
db
[prod:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/prod.pem
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# inventory.yml
all:
  children:
    web:
      hosts: { web1: { ansible_host: 10.0.1.10 }, web2: { ansible_host: 10.0.1.11 } }
      vars: { http_port: 8080 }
```

```yaml
# aws_ec2.yml  (dynamic inventory, plugin amazon.aws.aws_ec2)
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters: { tag:Env: prod, instance-state-name: running }
keyed_groups:
  - key: tags.Role
    prefix: role
hostnames: [private-ip-address]
compose:
  ansible_host: private_ip_address
```

Group/host vars: `group_vars/all.yml`, `group_vars/web.yml`, `host_vars/web1.yml`. Precedence (low→high): role defaults → inventory vars → group_vars → host_vars → play vars → role vars → set_fact → extra vars (`-e` always wins).

## Playbook

```yaml
- name: Configure web tier
  hosts: web
  become: true
  gather_facts: true
  serial: "25%"                 # rolling
  max_fail_percentage: 10
  vars:
    http_port: 8080
  vars_files: [vars/{{ env }}.yml]
  pre_tasks:
    - name: Deregister from ALB
      amazon.aws.elb_target:
        target_group_arn: "{{ tg_arn }}"
        target_id: "{{ ansible_ec2_instance_id }}"
        state: absent
      delegate_to: localhost
  roles:
    - role: common
    - role: nginx
      vars: { nginx_worker_processes: 4 }
  tasks:
    - name: Install packages
      ansible.builtin.package:
        name: [nginx, jq]
        state: present
      when: ansible_os_family == "RedHat"
      tags: [nginx]

    - name: Template config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: "0644"
        validate: nginx -t -c %s
        backup: true
      notify: Reload nginx

    - name: Ensure dirs
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: "0755"
      loop: [/var/www/app, /var/log/app]

    - name: Run migration once
      ansible.builtin.command: /opt/app/migrate
      args: { creates: /opt/app/.migrated }     # idempotency guard
      register: mig
      changed_when: "'applied' in mig.stdout"
      failed_when: mig.rc not in [0, 2]

    - name: Wait for port
      ansible.builtin.wait_for: { port: 8080, delay: 2, timeout: 60 }

    - name: Health check
      ansible.builtin.uri:
        url: "http://localhost:8080/health"
        status_code: 200
      retries: 5
      delay: 3
      register: hc
      until: hc.status == 200

    - block:
        - ansible.builtin.command: /opt/app/deploy
      rescue:
        - ansible.builtin.command: /opt/app/rollback
      always:
        - ansible.builtin.debug: { msg: "deploy finished on {{ inventory_hostname }}" }

  post_tasks:
    - name: Re-register to ALB
      amazon.aws.elb_target: { target_group_arn: "{{ tg_arn }}", target_id: "{{ ansible_ec2_instance_id }}", state: present }
      delegate_to: localhost

  handlers:
    - name: Reload nginx
      ansible.builtin.service: { name: nginx, state: reloaded }
```

## Common modules

| Area | Modules |
|---|---|
| Packages | `package`, `dnf`/`yum`, `apt`, `pip` |
| Files | `copy`, `template`, `file`, `lineinfile`, `blockinfile`, `replace`, `fetch`, `unarchive`, `get_url`, `stat`, `find`, `synchronize` |
| Services | `service`, `systemd` (`daemon_reload: true`, `enabled`, `state`) |
| Users | `user`, `group`, `authorized_key` |
| Commands | `command` (no shell), `shell` (pipes), `raw`, `script`; always add `creates`/`removes`/`changed_when` |
| Control | `debug`, `assert`, `fail`, `set_fact`, `include_tasks`, `import_tasks`, `include_role`, `meta: flush_handlers`, `pause`, `wait_for`, `wait_for_connection` |
| Cloud | `amazon.aws.ec2_instance`, `s3_object`, `ec2_security_group`, `elb_target`, `aws_ssm` lookup, `community.aws.*`, `kubernetes.core.k8s`, `helm` |
| Misc | `uri`, `cron`, `mount`, `sysctl`, `firewalld`, `seboolean`, `hostname`, `timezone`, `reboot` |

## Jinja2

```jinja
{{ var | default('x') }} {{ list | join(',') }} {{ dict | to_nice_json }} {{ s | upper }} {{ path | basename }}
{{ hostvars['db1'].ansible_default_ipv4.address }} {{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}
{% for h in groups['web'] %}server {{ hostvars[h].ansible_host }}:8080;{% endfor %}
{% if env == 'prod' %}worker_processes auto;{% else %}worker_processes 1;{% endif %}
{{ lookup('env', 'HOME') }} {{ lookup('file', 'id_rsa.pub') }} {{ lookup('amazon.aws.aws_ssm', '/app/db_pass', region='ap-south-1') }}
{{ ansible_facts['memtotal_mb'] }} {{ ansible_hostname }} {{ inventory_hostname }} {{ ansible_date_time.iso8601 }}
```

Loops: `loop`, `loop_control: { label: "{{ item.name }}", index_var: i }`, `with_items` (legacy), `with_dict`, `loop: "{{ dict | dict2items }}"`, `until/retries/delay`.

## Roles

```
roles/nginx/
├── defaults/main.yml   lowest-precedence vars
├── vars/main.yml       high precedence
├── tasks/main.yml
├── handlers/main.yml
├── templates/*.j2
├── files/
├── meta/main.yml       dependencies
└── molecule/           tests
```

## ansible.cfg essentials

```ini
[defaults]
inventory = inventories/prod/aws_ec2.yml
remote_user = ec2-user
host_key_checking = False
forks = 20
timeout = 30
retry_files_enabled = False
stdout_callback = yaml
callbacks_enabled = timer, profile_tasks
interpreter_python = auto_silent
vault_password_file = ~/.vault_pass
roles_path = roles:~/.ansible/roles
collections_path = collections
[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
[privilege_escalation]
become = True
become_method = sudo
```

## Connection via SSM (no SSH)

```yaml
ansible_connection: community.aws.aws_ssm
ansible_aws_ssm_bucket_name: my-ssm-transfer-bucket
ansible_aws_ssm_region: ap-south-1
```

## Interview one-liners

- **Idempotent**: modules check state before changing; `command`/`shell` aren't, so guard with `creates`, `changed_when`, or use a real module.
- **Handlers** run once at the end of the play (or on `meta: flush_handlers`) only if notified by a changed task.
- **Facts** gathered by `setup`; disable for speed (`gather_facts: false`) or cache (`fact_caching = jsonfile`).
- **Check mode** `--check --diff` = dry run; some tasks need `check_mode: false` to gather data.
- **Ansible vs Terraform**: Terraform provisions infrastructure (declarative, stateful); Ansible configures what's on it (procedural-ish, stateless). Together: Terraform creates instances, Ansible (or Packer+Ansible) configures. Prefer baking with Packer for immutability.
- **Ansible vs Chef/Puppet/Salt**: agentless, push-based, YAML; Puppet/Chef need agents and a master, pull-based, better for huge long-lived fleets with continuous enforcement.
- **AWX / Automation Platform (Tower)**: UI, RBAC, scheduling, credentials vault, job templates, surveys, callbacks.
- **Strategies**: `linear` (default), `free`, `serial` for rolling; `throttle` per task; `run_once`; `delegate_to`; `local_action`.
