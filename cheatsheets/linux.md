# Linux cheatsheet (annotated)

[← Cheatsheet index](README.md)

Format: command → what it does and when you reach for it.

## Files and directories

| Command | What it does / when to use |
|---|---|
| `ls -lah --sort=time \| head` | Long listing, human sizes, newest first (`-t` on macOS). Use to find what just changed in a directory. |
| `ls -laS` | Sort by size: what's big here. |
| `tree -L 2 -a` | Directory tree two levels deep including dotfiles. Quick orientation in a new repo or `/etc/app`. |
| `find / -name '*.log' -size +100M -mtime -1 2>/dev/null` | Logs over 100 MB modified in the last day. The disk-full detective. |
| `find . -type f -name '*.tmp' -mtime +7 -delete` | Delete temp files older than 7 days. Cron-friendly cleanup. |
| `find . -type d -empty -delete` | Remove empty directories. |
| `find /var/log -type f -exec gzip {} \;` | Run a command per match. |
| `stat file` | Size, permissions, owner, inode, and the three timestamps (access, modify, change). Use when "who touched this and when". |
| `file x` | Identifies file type by content (ELF binary, gzip, ASCII text). Use before you `cat` something unknown. |
| `which cmd` / `type cmd` | Which binary runs for a name; `type` also reveals aliases/functions/builtins. Use when "wrong version of terraform is running". |
| `cp -a src/ dst/` | Archive copy preserving permissions, ownership, timestamps, symlinks. |
| `rsync -avz --delete --progress src/ user@host:/dst/` | Efficient sync: only changed blocks, preserves attributes, `--delete` mirrors removals. Use for deployments and backups; add `-n` for dry run. |
| `ln -s /target /link` / `readlink -f /link` | Create a symlink; resolve where a link finally points. |
| `mkdir -p a/b/c` | Create nested directories, no error if existing. |
| `mktemp` / `mktemp -d` | Safe unique temp file/dir. Always use in scripts instead of fixed `/tmp/x` names. |
| `tar czf out.tgz dir/` / `tar xzf out.tgz -C /dst` / `tar tzf out.tgz` | Create / extract into a directory / list. `c`reate `x`tract `t`list, `z` gzip, `j` bz2, `J` xz, `f` file. |
| `du -sh * \| sort -h \| tail` | Size per entry in the current dir, biggest last. Drill down to find what fills the disk. |
| `du -xh --max-depth=1 / \| sort -h` | Same at root, staying on one filesystem (`-x`) so mounts don't confuse it. |
| `df -hT` | Free space per filesystem with type. |
| `df -i` | Free inodes. "No space left on device" with free GB = inodes exhausted by millions of small files. |
| `lsof +D /path` | Processes with files open under a path. Use before unmounting or when a delete didn't free space. |
| `lsof \| grep deleted` | Deleted files still held open by a process (space not released). Restart the process or truncate via `/proc/PID/fd/N`. |
| `truncate -s 0 big.log` | Empty a file in place. Use instead of `rm` on a log a daemon holds open. |

## Permissions and ownership

| Command | What it does / when to use |
|---|---|
| `chmod 644 f` / `chmod 755 d` | rw-r--r-- for files, rwxr-xr-x for dirs/scripts. r=4 w=2 x=1 per owner/group/other. |
| `chmod u+x s.sh` | Make a script executable for the owner. |
| `chmod -R g+rw dir` | Recursive group read/write. |
| `chmod g+s dir` | Setgid on a directory: new files inherit the directory's group. For shared team folders. |
| `chown user:group f` / `chown -R app:app /opt/app` | Change owner (recursively). Fix "permission denied" when a service runs as `app`. |
| `umask 022` | Default permission mask for new files (022 → 644 files, 755 dirs). |
| `ls -l /usr/bin/passwd` → `-rwsr-xr-x` | The `s` is setuid: runs as the file owner (root). Audit these. |
| `getfacl f` / `setfacl -m u:alice:rw f` | View/add ACLs beyond owner/group/other. |
| `sudo -l` | What you're allowed to run with sudo. |
| `sudo -u app bash` | Shell as the service user to reproduce its permission errors. |
| `visudo` | Safely edit sudoers (syntax-checked). Put rules in `/etc/sudoers.d/`. |
| `lsattr f` / `chattr +i f` | Extended attributes; `+i` makes a file immutable even for root. Protects critical configs. |

## Users and groups

| Command | What it does / when to use |
|---|---|
| `id` / `whoami` / `groups` | Who you are and your groups (e.g. are you in `docker`?). |
| `who` / `w` / `last` / `lastlog` | Logged-in users, their activity, login history, last login per user. Audit basics. |
| `useradd -m -s /bin/bash -G docker,sudo alice` | Create user with home, shell, supplementary groups. |
| `passwd alice` | Set password. |
| `usermod -aG docker alice` | Append a group (`-a` matters; without it you replace all groups). |
| `userdel -r alice` | Remove user and home. |
| `groupadd ops` / `gpasswd -a alice ops` | Create group / add member. |
| `cat /etc/passwd /etc/group` | Users (`name:x:uid:gid:comment:home:shell`) and groups. |
| `chage -l alice` / `chage -M 90 alice` | Password ageing info / force 90-day expiry (compliance). |

## Processes

| Command | What it does / when to use |
|---|---|
| `ps aux --sort=-%mem \| head` | Top memory consumers. |
| `ps -ef --forest` | Process tree showing parent/child. Use to see what spawned what. |
| `ps -o pid,ppid,etime,cmd -p PID` | Chosen columns for one process (`etime` = how long running). |
| `pgrep -af nginx` | Find PIDs by name with full command line. |
| `pkill -f 'python app.py'` | Kill by matching command line. |
| `kill -15 PID` / `kill -9 PID` | SIGTERM (ask nicely, lets it clean up) / SIGKILL (immediate, no cleanup). Always try 15 first. |
| `killall node` | Kill all processes by name. |
| `top -o %CPU` / `htop` / `atop` | Live process views; `atop` also logs history. |
| `nice -n 10 cmd` / `renice -n 5 -p PID` | Start or adjust CPU priority (higher nice = lower priority). For batch jobs on shared hosts. |
| `nohup cmd &> out.log &` | Run detached from the terminal, surviving logout. |
| `disown` / `jobs` / `fg` / `bg` / `Ctrl-Z` | Job control: suspend, background, foreground. |
| `tmux new -s x` / `tmux attach -t x` / `screen -S x` | Persistent terminal sessions. Use for long-running operations over SSH. |
| `strace -f -p PID` | Trace system calls of a running process. Use when a process hangs: see which file/socket it's stuck on. `-e trace=network` to see only network calls. |
| `ltrace` | Trace library calls. |
| `lsof -p PID` | Open files/sockets of a process. |
| `lsof -i :8080` | Who is listening on or connected to port 8080. |
| `lsof -u app` | Everything a user has open. |
| `/proc/PID/cmdline`, `environ`, `status`, `fd/`, `limits`, `cwd` | Raw process info: exact command, environment (secrets a process sees), memory, open FDs, ulimits, working dir. `cat /proc/PID/environ \| tr '\0' '\n'`. |
| `ulimit -n` / `ulimit -a` | Open-file limit / all limits for this shell. "Too many open files" → raise in `/etc/security/limits.conf` or systemd `LimitNOFILE`. |
| `uptime` / `cat /proc/loadavg` | Load averages (1/5/15 min). Compare to `nproc`: load > cores means waiting. |
| `nproc` / `lscpu` | CPU count / topology. |
| `free -h` | Memory: look at "available", not "free" (cache is reclaimable). |
| `vmstat 1 5` | Per-second system view: `r` runnable, `b` blocked on I/O, `si/so` swap in/out, `us/sy/id/wa` CPU split. The best single triage command. |
| `iostat -xz 1` | Disk latency (`await`) and utilisation per device. Use when `wa` is high. |
| `sar -u 1 5` / `mpstat -P ALL` | CPU history / per-core usage (is one core pegged?). |
| `dmesg -T \| tail` | Kernel ring buffer with timestamps: OOM kills, disk errors, NIC resets. |
| `dmesg \| grep -i -E 'oom\|killed'` | Did the OOM killer fire, and on what. |

## systemd and journald

| Command | What it does / when to use |
|---|---|
| `systemctl status nginx` | Running? Since when? Last log lines. The first command for any service issue. |
| `systemctl start/stop/restart/reload nginx` | Reload re-reads config without dropping connections (if the service supports it); restart does. |
| `systemctl enable --now nginx` | Start now and at boot in one command. |
| `systemctl disable/mask nginx` | Disable at boot; mask makes it impossible to start even manually. |
| `systemctl list-units --type=service --state=failed` / `systemctl --failed` | Everything that failed since boot. |
| `systemctl daemon-reload` | Re-read unit files after editing them. Forgetting this is why "my change did nothing". |
| `systemctl cat nginx` | Show the effective unit file including drop-ins. |
| `systemctl show nginx -p MainPID` | One property, e.g. the PID to feed `strace`. |
| `systemctl edit nginx` | Creates an override drop-in in `/etc/systemd/system/nginx.service.d/` instead of editing the vendor file (survives package upgrades). |
| `systemctl list-timers` | systemd timers (modern cron). |
| `systemd-analyze critical-chain svc` | Boot ordering: why a service started late or before its dependency. |
| `journalctl -u nginx -f` | Follow a service's logs. |
| `journalctl -u nginx --since '1 hour ago' --until '10 min ago'` | Time window. |
| `journalctl -p err -b` | Errors and worse since this boot. `-b -1` = previous boot (why did it crash before reboot). |
| `journalctl -k` | Kernel messages (same as dmesg, with persistence). |
| `journalctl --disk-usage` / `journalctl --vacuum-time=7d` | Journal size / trim to 7 days. |
| `journalctl -o json-pretty -u app \| jq` | Structured output for scripting. |

```ini
# /etc/systemd/system/app.service – a service unit, annotated
[Unit]
Description=Payments API
After=network-online.target        # order: start after network is really up
Wants=network-online.target        # and pull that target in
[Service]
Type=simple                        # process stays in foreground (notify/forking for others)
User=app                           # never root
WorkingDirectory=/opt/app
EnvironmentFile=/etc/app/env       # KEY=VALUE file; keep secrets out of the unit
ExecStart=/opt/app/bin/api --port 8080
Restart=on-failure                 # auto-restart on non-zero exit
RestartSec=5
LimitNOFILE=65536                  # raise open-files limit for this service only
[Install]
WantedBy=multi-user.target         # "enable" links it here so it starts at boot
```

## Packages

| Command | What it does / when to use |
|---|---|
| `dnf install -y pkg` / `dnf update -y` / `dnf remove pkg` | RHEL/AL2023/Fedora install/update/remove. |
| `dnf search x` / `dnf info x` / `dnf list installed` | Find, describe, list. |
| `dnf provides */bin/ss` | Which package ships a file. Use when a command is missing. |
| `dnf history` / `dnf history undo 12` | Transaction log / roll back a transaction. |
| `rpm -qa \| grep x` / `rpm -ql pkg` / `rpm -qf /path` | Installed packages / files in a package / which package owns a file. |
| `apt update && apt install -y pkg` | Debian/Ubuntu: refresh index then install (always update first). |
| `apt upgrade` / `apt remove\|purge pkg` | Upgrade all / remove (purge also deletes config). |
| `apt search x` / `apt show x` / `apt list --installed` | Find, describe, list. |
| `dpkg -l \| grep x` / `dpkg -L pkg` / `dpkg -S /path` | Installed / files in package / owner of a file. |
| `apt-mark hold pkg` | Pin a version so upgrades don't change it (kernel, docker). |
| `yum install -y pkg` / `amazon-linux-extras list` | AL2 equivalents. |

## Text processing

| Command | What it does / when to use |
|---|---|
| `grep -rniE 'error\|fail' /var/log --include='*.log'` | Recursive, case-insensitive, line numbers, extended regex, only .log files. |
| `grep -v '^#' f \| grep -v '^$'` | Strip comments and blank lines from a config. |
| `grep -c` / `-o` / `-l` / `-A3 -B1` / `-F` | Count matches / print only the match / only filenames / context lines / fixed string (no regex, faster). |
| `awk '{print $1,$NF}' f` | First and last field. Default separator is whitespace. |
| `awk -F: '$3>=1000{print $1}' /etc/passwd` | Custom separator and a condition: real users. |
| `awk '/ERROR/{c++} END{print c}'` | Count matching lines. |
| `awk '{s+=$5} END{print s/NR}'` | Average of column 5. |
| `awk '!seen[$0]++' f` | Deduplicate preserving order (unlike `sort -u`). |
| `sed -n '10,20p' f` | Print lines 10–20. |
| `sed -i.bak 's/old/new/g' f` | In-place replace with a backup. Use `-E` for extended regex, `\1` for capture groups. |
| `sed -i '/pattern/d' f` | Delete matching lines. |
| `sed '1i header' f` / `sed '$a footer' f` | Insert before line 1 / append after last line. |
| `cut -d, -f1,3 f` / `cut -c1-10` | Fields by delimiter / characters by position. |
| `tr 'a-z' 'A-Z'` / `tr -d '\r'` / `tr -s ' '` | Translate case / delete carriage returns (Windows files) / squeeze repeated spaces. |
| `sort -u` / `sort -t, -k2,2n` / `sort -h` / `sort -rn` | Unique / by field 2 numerically / human sizes / reverse numeric. |
| `uniq -c` / `uniq -d` | Count adjacent duplicates (sort first) / show only duplicates. `sort \| uniq -c \| sort -rn \| head` = top-N anything. |
| `wc -l` / `head -n 20` / `tail -n 50` | Count lines / first / last. |
| `tail -F f` | Follow a file across rotations (capital F). `less +F` to follow with search. |
| `diff -u a b` / `diff -r d1 d2` | Unified diff / recursive directory diff. |
| `comm -12 <(sort a) <(sort b)` | Lines common to both files (sorted input). `-23` = only in a. |
| `paste -d, a b` / `join` / `column -t` | Merge files side by side / relational join / align columns for readability. |
| `xargs -n1 -P4 -I{} cmd {}` | Run a command per input line, 4 in parallel. Use with `find -print0 \| xargs -0` for safe filenames. |
| `tee out.log` / `tee -a` / `cmd 2>&1 \| tee log` | Write to file and stdout at once / append / capture stderr too. |
| `cmd > /dev/null 2>&1` | Silence everything. |
| `jq '.a.b[] \| select(.x=="y") \| .id' f.json` | Query JSON. `-r` raw strings, `-c` compact, `keys`, `length`. The AWS CLI's best friend. |
| `yq '.spec.replicas' f.yaml` | jq for YAML. |
| `base64 -w0 f` / `base64 -d` | Encode (no wrapping) / decode. Kubernetes Secrets. |
| `sha256sum -c sums.txt` / `md5sum` | Verify checksums of downloads. |
| `xxd` / `od -c` / `strings bin \| grep x` | Hex dump / character dump (find hidden `\r`) / printable strings in a binary. |
| `envsubst < template > out` | Substitute `$VARS` in a file. Poor man's templating for manifests. |
| `printf '%s\n' "${arr[@]}"` | Print array elements safely (unlike `echo`). |
| `date -u +%FT%TZ` / `date -d @1700000000` / `date -d '2 days ago'` | ISO-8601 UTC now / epoch to date / relative dates (GNU; macOS uses `-v-2d`). |

## Disks, filesystems, LVM

| Command | What it does / when to use |
|---|---|
| `lsblk -f` | Block devices with filesystem type, UUID, mount point. Start here after attaching an EBS volume (shows as `nvme1n1`). |
| `blkid` / `fdisk -l` / `parted /dev/nvme1n1 print` | UUIDs / partition tables. |
| `mkfs.xfs /dev/nvme1n1` / `mkfs.ext4 -L data /dev/xvdf` | Format a new volume (destroys data). XFS is the AL2023 default. |
| `mount /dev/nvme1n1 /data` / `umount /data` | Mount / unmount. `mount -a` mounts everything in fstab (test fstab edits before rebooting). |
| `findmnt` | Tree of mounts with options. |
| `/etc/fstab`: `UUID=xxxx /data xfs defaults,nofail 0 2` | Persistent mount by UUID (device names change on EC2); `nofail` so a missing EBS volume doesn't block boot. |
| `growpart /dev/nvme0n1 1` | Grow partition 1 to fill the disk after resizing the EBS volume (root volumes). |
| `xfs_growfs /data` / `resize2fs /dev/xvdf` | Grow the filesystem online after `modify-volume`/`growpart`. |
| `pvcreate` / `vgcreate vg0 /dev/xvdg` / `lvcreate -L 20G -n lv0 vg0` / `lvextend -r -L +10G /dev/vg0/lv0` | LVM: physical volume → volume group → logical volume; `-r` resizes the filesystem too. `pvs vgs lvs` to inspect. |
| `swapon --show` / `fallocate -l 2G /swap && mkswap /swap && swapon /swap` | Check / add a swap file (buffer against OOM on small instances). |
| `fsck -y /dev/xvdf` (unmounted) / `xfs_repair` | Repair a filesystem. |
| `iotop` / `ionice` | Per-process I/O / lower a process's I/O priority. |

## Kernel, boot, scheduling, misc

| Command | What it does / when to use |
|---|---|
| `uname -a` / `cat /etc/os-release` / `hostnamectl` | Kernel, distro, hostname/VM info. |
| `timedatectl` / `chronyc sources` | Time zone and NTP sync status. Clock skew breaks TLS and SigV4 signatures. |
| `sysctl -a \| grep net.ipv4.ip_forward` | Read a kernel tunable. |
| `sysctl -w net.core.somaxconn=4096` | Set at runtime; persist in `/etc/sysctl.d/99-custom.conf` and `sysctl -p`. Common: `vm.max_map_count` for Elasticsearch, `net.ipv4.ip_forward` for routers/K8s. |
| `lsmod` / `modprobe br_netfilter` | Loaded kernel modules / load one (Kubernetes needs `br_netfilter`, `overlay`). |
| `cat /proc/meminfo` / `/proc/cpuinfo` / `dmidecode -t system` | Hardware details. |
| `crontab -l` / `crontab -e` / `/etc/cron.d/` / `/var/log/cron` | List / edit user cron / system cron files / cron log. Format `m h dom mon dow cmd`; `@reboot`. Wrap jobs in `flock -n` to avoid overlaps. |
| `echo $?` | Exit code of the last command (0 = success). |
| `bash -x script.sh` / `set -x` | Trace every command as it runs. The debugging switch for shell scripts. |
| `shellcheck script.sh` | Static analysis for shell scripts. Run in pre-commit. |
| `history \| tail` / `!!` / `!$` / `Ctrl-R` | Recent commands / repeat last / last argument / reverse search. |
| `export PATH=$PATH:/opt/bin` / `source ~/.bashrc` / `env` / `unset X` | Environment management. |
| `cloud-init status` / `cat /var/log/cloud-init-output.log` | Did user data finish, and what did it print. The EC2 bootstrap debugger. |
| `cloud-init clean` | Reset so cloud-init runs again on next boot (for AMI building). |
| `TOKEN=$(curl -sX PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 60'); curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id` | IMDSv2 two-step: get a token, then query metadata (instance ID, IAM role creds, user data). IMDSv1 one-step GET should be disabled. |

## Where the logs are

| Path | What |
|---|---|
| `/var/log/messages` (RHEL) / `/var/log/syslog` (Debian) | General system log |
| `/var/log/secure` / `/var/log/auth.log` | SSH logins, sudo, PAM |
| `/var/log/cloud-init-output.log` | EC2 user-data output |
| `dmesg -T`, `/var/log/dmesg` | Kernel: OOM, hardware, NIC |
| `/var/log/nginx/`, `/var/log/httpd/` | Web servers |
| `/var/log/audit/audit.log` | auditd / SELinux denials; query with `ausearch -m avc -ts recent` |
| `/var/log/amazon/ssm/`, `/opt/aws/amazon-cloudwatch-agent/logs/` | AWS agents |
| `logrotate -d /etc/logrotate.conf` | Dry-run rotation to debug a config |

## Performance triage order (USE method)

1. `uptime` – load vs `nproc`
2. `dmesg -T | tail` – kernel errors, OOM
3. `vmstat 1` – r (runnable), b (blocked), si/so (swapping), us/sy/id/wa
4. `mpstat -P ALL 1` – one hot core?
5. `pidstat 1` – which process
6. `iostat -xz 1` – disk await / %util
7. `free -h` – memory and cache
8. `sar -n DEV 1` / `ss -s` – network throughput / socket states
9. `top` / `htop` – confirm and act
