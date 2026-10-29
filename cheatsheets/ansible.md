# Ansible cheatsheet (annotated)

[← Cheatsheet index](README.md)

Ansible = agentless configuration management over SSH (or WinRM / AWS SSM): YAML playbooks of idempotent modules. In an immutable-image world it runs inside the Packer build; in a mutable world it configures running fleets.

Format: command → what it does and when you reach for it.

## CLI

- `ansible --version`

  Version plus which `ansible.cfg` and Python are in use. First line of any bug report.

- `ansible-config dump --only-changed`

  Settings that differ from defaults. Debug "why is it using the wrong inventory".

- `ansible-config init --disabled > ansible.cfg`

  Generate a fully commented config to start from.

- `ansible all -i inventory.ini -m ping`

  Ad hoc: can Ansible reach and run Python on every host. The connectivity smoke test.

- `ansible web -m shell -a 'uptime' -b`

  Ad hoc command on a group, with `become` (sudo). Quick fleet-wide checks.

- `ansible db -m setup -a 'filter=ansible_distribution*'`

  Gather facts (filtered). Use to see what variables you can branch on.

- `ansible all -m copy -a 'src=./f dest=/tmp/f mode=0644' -b`

  Ad hoc module with arguments. Any module works this way.

- `ansible all -m service -a 'name=nginx state=restarted' -b`

  Restart a service everywhere.

- `ansible-inventory -i inventory --list` / `--graph` / `--host web1`

  Show the resolved inventory as JSON / as a tree / variables for one host. Debug dynamic inventory.

- `ansible-playbook site.yml -i inventories/prod`

  Run a playbook against an inventory.

- `--limit web`

  Restrict to a group/host pattern (`web:&prod`, `!db`).

- `--tags nginx` / `--skip-tags slow`

  Run or skip tagged tasks.

- `--check --diff`

  Dry run showing what would change and file diffs. Run before every prod change.

- `-e 'env=prod' -e @vars.json`

  Extra vars (highest precedence) inline or from a file.

- `--start-at-task 'Install nginx'`

  Resume from a task after fixing a failure mid-play.

- `--step`

  Confirm each task interactively.

- `-vvv`

  Verbose: shows the SSH command and module output. `-vvvv` adds connection debugging.

- `--list-tasks` / `--list-hosts` / `--syntax-check`

  Preview what would run / on which hosts / parse only. CI uses `--syntax-check`.

- `ansible-vault create secrets.yml` / `edit` / `view` / `encrypt` / `decrypt` / `rekey`

  Manage encrypted variable files committed to git.

- `ansible-vault encrypt_string 'pw' --name db_pass`

  Encrypt a single value to paste into a plaintext vars file.

- `--ask-vault-pass` / `--vault-password-file ~/.vault_pass` / `--vault-id prod@prompt`

  Supply the vault password interactively, from a file (CI), or per-environment vault IDs.

- `ansible-galaxy init roles/nginx`

  Scaffold a role directory structure.

- `ansible-galaxy install -r requirements.yml` / `collection install amazon.aws community.general`

  Install roles/collections pinned in a requirements file.

- `ansible-doc copy` / `ansible-doc -l | grep aws` / `ansible-doc -t lookup aws_ssm`

  Module docs offline / list modules / docs for a lookup plugin. Use instead of guessing arguments.

- `ansible-lint site.yml`

  Best-practice linter (no `shell` without `changed_when`, FQCN module names, etc.). CI gate.

- `molecule test`

  Spin up a container/instance, apply the role, verify, check idempotence, destroy. Role testing.

- `ansible-pull -U git@repo site.yml`

  Pull mode: the host fetches the repo and applies to itself (cron). For hosts the controller can't reach.

## Inventory

```ini
# inventory.ini – static INI
[web]
web1 ansible_host=10.0.1.10          # alias → real address
web2 ansible_host=10.0.1.11
[db]
db1 ansible_host=10.0.2.10 ansible_user=admin
[prod:children]                      # group of groups
web
db
[prod:vars]                          # variables for every host in prod
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/prod.pem
ansible_python_interpreter=/usr/bin/python3
```

```yaml
# aws_ec2.yml – dynamic inventory from EC2 tags (plugin amazon.aws.aws_ec2); file name must end in aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
filters: { tag:Env: prod, instance-state-name: running }   # only running prod instances
keyed_groups:
  - key: tags.Role                   # creates groups role_web, role_db from the Role tag
    prefix: role
hostnames: [private-ip-address]      # how hosts are named
compose:
  ansible_host: private_ip_address   # connect over private IPs (via bastion/VPN/SSM)
```

Group/host variables live in `group_vars/all.yml`, `group_vars/web.yml`, `host_vars/web1.yml` next to the inventory.

Variable precedence (low → high): role `defaults/` → inventory vars → `group_vars/all` → parent group → child group → `host_vars` → play `vars`/`vars_files` → role `vars/` → block/task vars → `set_fact`/`register` → `-e` extra vars (always win).

## Playbook (annotated)

```yaml
- name: Configure web tier
  hosts: web                          # inventory group/pattern
  become: true                        # sudo for all tasks
  gather_facts: true                  # collect ansible_* facts (set false for speed when unused)
  serial: "25%"                       # rolling: 25% of hosts at a time; a number also works
  max_fail_percentage: 10             # abort the play if >10% of hosts fail in a batch
  vars:
    http_port: 8080
  vars_files: [vars/{{ env }}.yml]    # per-environment variables
  pre_tasks:
    - name: Deregister from ALB        # take the host out of rotation before changing it
      amazon.aws.elb_target:
        target_group_arn: "{{ tg_arn }}"
        target_id: "{{ ansible_ec2_instance_id }}"
        state: absent
      delegate_to: localhost          # run the AWS API call from the controller, not the host
  roles:
    - role: common
    - role: nginx
      vars: { nginx_worker_processes: 4 }
  tasks:
    - name: Install packages
      ansible.builtin.package:        # generic package module picks dnf/apt per OS
        name: [nginx, jq]
        state: present                # idempotent: no-op if already installed
      when: ansible_os_family == "RedHat"
      tags: [nginx]

    - name: Template config
      ansible.builtin.template:
        src: nginx.conf.j2            # Jinja2 template in the role's templates/
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: "0644"
        validate: nginx -t -c %s      # test the rendered file before replacing the real one
        backup: true                  # keep a timestamped copy of the old file
      notify: Reload nginx            # trigger the handler only if the file changed

    - name: Ensure dirs
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: "0755"
      loop: [/var/www/app, /var/log/app]

    - name: Run migration once
      ansible.builtin.command: /opt/app/migrate
      args: { creates: /opt/app/.migrated }      # skip if this file exists → idempotent command
      register: mig                              # capture output/rc
      changed_when: "'applied' in mig.stdout"    # report changed only when it did something
      failed_when: mig.rc not in [0, 2]          # treat rc 2 as success

    - name: Wait for port
      ansible.builtin.wait_for: { port: 8080, delay: 2, timeout: 60 }   # block until the app listens

    - name: Health check
      ansible.builtin.uri:
        url: "http://localhost:8080/health"
        status_code: 200
      register: hc
      retries: 5
      delay: 3
      until: hc.status == 200          # retry loop

    - block:                           # try / rescue / always
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
    - name: Reload nginx               # runs once at the end of the play if notified
      ansible.builtin.service: { name: nginx, state: reloaded }
```

## Modules by job

- **Install software**

  `package` (OS-agnostic), `dnf`/`yum`, `apt` (`update_cache: true`), `pip`

- **Manage files**

  `copy` (static), `template` (Jinja2), `file` (perms/dirs/links), `lineinfile` (one line, regex), `blockinfile` (marked block), `replace`, `fetch` (pull from host), `unarchive`, `get_url`, `stat`, `find`, `synchronize` (rsync)

- **Services**

  `service`, `systemd` (`daemon_reload: true` after unit changes, `enabled`, `state`)

- **Users**

  `user`, `group`, `authorized_key`

- **Run commands**

  `command` (no shell features, safest), `shell` (pipes/redirects), `raw` (no Python needed), `script` (upload+run). Always add `creates`/`removes`/`changed_when` so they're idempotent.

- **Flow control**

  `debug`, `assert` (fail early on bad vars), `fail`, `set_fact`, `include_tasks` (dynamic) / `import_tasks` (static), `include_role`, `meta: flush_handlers`, `pause`, `wait_for`, `wait_for_connection` (after reboot)

- **AWS**

  `amazon.aws.ec2_instance`, `s3_object`, `ec2_security_group`, `elb_target`, `aws_ssm` lookup; `community.aws.*`

- **Kubernetes**

  `kubernetes.core.k8s` (apply manifests), `kubernetes.core.helm`

- **System**

  `cron`, `mount`, `sysctl`, `firewalld`, `seboolean`, `hostname`, `timezone`, `reboot` (waits for the host to return)

- **HTTP**

  `uri` (health checks, API calls)

## Jinja2 in templates and tasks

- `{{ var | default('x') }}`

  Fallback if undefined.

- `{{ list | join(',') }}` / `{{ dict | to_nice_json }}` / `{{ s | upper }}` / `{{ path | basename }}`

  Common filters.

- `{{ hostvars['db1'].ansible_default_ipv4.address }}`

  Another host's fact (needs facts gathered for that host).

- `{{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}`

  IPs of all hosts in a group: build backend lists.

- `{% for h in groups['web'] %}server {{ hostvars[h].ansible_host }}:8080;{% endfor %}`

  Loop in a template (nginx upstream).

- `{% if env == 'prod' %}...{% else %}...{% endif %}`

  Conditional content.

- `{{ lookup('env', 'HOME') }}` / `{{ lookup('file', 'id_rsa.pub') }}`

  Controller-side lookups.

- `{{ lookup('amazon.aws.aws_ssm', '/app/db_pass', region='ap-south-1') }}`

  Pull a secret from SSM at run time instead of storing it in git.

- `{{ ansible_facts['memtotal_mb'] }}` / `{{ ansible_hostname }}` / `{{ inventory_hostname }}` / `{{ ansible_date_time.iso8601 }}`

  Facts and built-ins.

- `loop:` + `loop_control: { label: "{{ item.name }}", index_var: i }`

  Loop with readable output and an index. `loop: "{{ dict | dict2items }}"` to loop a map.

## Role layout

```
roles/nginx/
├── defaults/main.yml   # tunables with lowest precedence (callers override freely)
├── vars/main.yml       # internal constants, high precedence (hard to override)
├── tasks/main.yml      # entry point
├── handlers/main.yml
├── templates/*.j2
├── files/              # static files for copy
├── meta/main.yml       # dependencies on other roles, galaxy metadata
└── molecule/           # tests
```

## ansible.cfg essentials

```ini
[defaults]
inventory = inventories/prod/aws_ec2.yml   # default inventory
remote_user = ec2-user
host_key_checking = False                  # for ephemeral cloud hosts (accept the trade-off)
forks = 20                                 # parallel hosts
timeout = 30
retry_files_enabled = False                # don't litter .retry files
stdout_callback = yaml                     # readable output
callbacks_enabled = timer, profile_tasks   # show where time goes
interpreter_python = auto_silent
vault_password_file = ~/.vault_pass
roles_path = roles:~/.ansible/roles
collections_path = collections
[ssh_connection]
pipelining = True                          # fewer SSH round trips, big speed-up (needs !requiretty in sudoers)
ssh_args = -o ControlMaster=auto -o ControlPersist=60s   # reuse SSH connections
[privilege_escalation]
become = True
become_method = sudo
```

## Connect over SSM (no SSH ports)

```yaml
ansible_connection: community.aws.aws_ssm
ansible_aws_ssm_bucket_name: my-ssm-transfer-bucket   # file transfer goes via S3
ansible_aws_ssm_region: ap-south-1
```
Hosts need the SSM agent and `AmazonSSMManagedInstanceCore`; the controller needs `ssm:StartSession` and S3 access. Every session is in CloudTrail.

## Interview one-liners

- **Idempotent**: modules check state before acting; `command`/`shell` don't, so guard with `creates`, `changed_when`, or use a real module.
- **Handlers** run once at the end of the play, only if notified by a changed task; `meta: flush_handlers` runs them now.
- **Facts** come from `setup`; disable (`gather_facts: false`) or cache (`fact_caching = jsonfile`) for speed.
- **Check mode** `--check --diff` = dry run; some tasks need `check_mode: false` to still gather data.
- **Ansible vs Terraform**: Terraform provisions (declarative, stateful); Ansible configures (procedural-ish, stateless). Prefer Packer+Ansible to bake, Terraform to deploy.
- **Ansible vs Puppet/Chef**: agentless push vs agent-based pull with continuous enforcement.
- **Strategies**: `linear` (default, task by task across hosts), `free` (hosts run ahead independently), `serial` for rolling, `throttle` per task, `run_once`, `delegate_to`.
