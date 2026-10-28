# Linux cheatsheet (DevOps)

[← Cheatsheet index](README.md)

## Navigation and files

```bash
pwd; cd -; cd ~; pushd dir; popd
ls -lah --sort=time | head          # newest first (-t on BSD)
ls -laS                              # by size
tree -L 2 -a
find / -name '*.log' -size +100M -mtime -1 2>/dev/null       # big logs changed in last day
find . -type f -name '*.tmp' -mtime +7 -delete
find . -type d -empty -delete
find /var/log -type f -exec gzip {} \;
locate nginx.conf; updatedb
stat file; file file; which cmd; type cmd; whereis cmd
cp -a src/ dst/                      # preserve perms/times
rsync -avz --delete --progress src/ user@host:/dst/
rsync -avz -e "ssh -i key.pem" ...
ln -s /target /link; readlink -f /link
mkdir -p a/b/c; mktemp; mktemp -d
tar czf out.tgz dir/; tar xzf out.tgz -C /dst; tar tzf out.tgz   # c=create x=extract t=list z=gzip j=bz2 J=xz
zip -r out.zip dir/; unzip -l out.zip
du -sh * | sort -h | tail; du -xh --max-depth=1 / | sort -h     # what's eating disk
df -hT; df -i                        # inodes exhausted = "No space left" with free space
lsof +D /path; lsof | grep deleted   # deleted-but-open files holding disk
truncate -s 0 big.log                # empty a file held open (don't rm)
```

## Permissions and ownership

```bash
chmod 644 f; chmod 755 d; chmod u+x s.sh; chmod -R g+rw dir; chmod g+s dir   # setgid dir
chown user:group f; chown -R app:app /opt/app
umask 022
# r=4 w=2 x=1 ; 4000 setuid 2000 setgid 1000 sticky
ls -l /usr/bin/passwd   # -rwsr-xr-x = setuid
getfacl f; setfacl -m u:alice:rw f
sudo -l; sudo -u app bash; visudo    # /etc/sudoers.d/
lsattr f; chattr +i f                # immutable
```

## Users and groups

```bash
id; whoami; groups; who; w; last; lastlog
useradd -m -s /bin/bash -G docker,sudo alice; passwd alice; usermod -aG docker alice; userdel -r alice
groupadd ops; gpasswd -a alice ops
cat /etc/passwd /etc/group /etc/shadow
chage -l alice; chage -M 90 alice
```

## Processes

```bash
ps aux --sort=-%mem | head; ps -ef --forest; ps -o pid,ppid,etime,cmd -p PID
pgrep -af nginx; pkill -f 'python app.py'; kill -15 PID; kill -9 PID; killall node
top -o %CPU; htop; atop
nice -n 10 cmd; renice -n 5 -p PID
nohup cmd &> out.log &; disown; jobs; fg; bg; ctrl-z
setsid cmd; screen -S x; tmux new -s x; tmux attach -t x
strace -f -p PID; strace -e trace=network cmd; ltrace
lsof -p PID; lsof -i :8080; lsof -u app
/proc/PID/{cmdline,environ,status,fd,limits}
ulimit -n; ulimit -a; /etc/security/limits.conf
uptime; cat /proc/loadavg; nproc; lscpu; free -h; vmstat 1 5; iostat -xz 1; sar -u 1 5; mpstat -P ALL
dmesg -T | tail; dmesg | grep -i -E 'oom|killed'      # OOM killer
```

## systemd

```bash
systemctl status nginx; systemctl start|stop|restart|reload|enable|disable|mask nginx
systemctl enable --now nginx
systemctl list-units --type=service --state=failed; systemctl --failed
systemctl daemon-reload                 # after editing unit files
systemctl cat nginx; systemctl show nginx -p MainPID
systemctl edit nginx                    # drop-in override in /etc/systemd/system/nginx.service.d/
systemctl list-timers; systemctl isolate multi-user.target
journalctl -u nginx -f; journalctl -u nginx --since '1 hour ago' --until '10 min ago'
journalctl -p err -b; journalctl -b -1; journalctl --disk-usage; journalctl --vacuum-time=7d
journalctl -o json-pretty -u app | jq
```

Unit file skeleton:
```ini
# /etc/systemd/system/app.service
[Unit]
Description=Payments API
After=network-online.target
Wants=network-online.target
[Service]
Type=simple
User=app
WorkingDirectory=/opt/app
EnvironmentFile=/etc/app/env
ExecStart=/opt/app/bin/api --port 8080
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
```

## Package management

```bash
# RHEL / AL2023 / Fedora
dnf install -y pkg; dnf update -y; dnf remove pkg; dnf search x; dnf info x; dnf list installed; dnf provides */bin/ss
dnf history; dnf history undo 12; dnf clean all; rpm -qa | grep x; rpm -ql pkg; rpm -qf /path
# Debian / Ubuntu
apt update && apt install -y pkg; apt upgrade; apt remove|purge pkg; apt search x; apt show x; apt list --installed
dpkg -l | grep x; dpkg -L pkg; dpkg -S /path; apt-mark hold pkg
# AL2
yum install -y pkg; amazon-linux-extras list
# Others
pip install -r requirements.txt; pipx; npm i -g x; snap list
```

## Text processing

```bash
grep -rniE 'error|fail' /var/log --include='*.log'; grep -v '^#' f | grep -v '^$'; grep -c; grep -o; grep -A3 -B1; grep -l; grep -F
awk '{print $1,$NF}' f; awk -F: '$3>=1000{print $1}' /etc/passwd; awk '/ERROR/{c++} END{print c}'; awk '{s+=$5} END{print s/NR}'
awk 'NR>1 && NR<=10'; awk '!seen[$0]++' f        # dedupe keep order
sed -n '10,20p' f; sed -i.bak 's/old/new/g' f; sed -i '/pattern/d' f; sed -E 's/([0-9]+)/<\1>/'; sed '1i header' f; sed '$a footer'
cut -d, -f1,3 f; cut -c1-10 f; tr 'a-z' 'A-Z'; tr -d '\r'; tr -s ' '
sort -u; sort -t, -k2,2n; sort -h; sort -rn | head; uniq -c; uniq -d
wc -l; head -n 20; tail -n 50; tail -F f (follows rotation); less +F f
diff -u a b; diff -r dir1 dir2; comm -12 <(sort a) <(sort b); vimdiff
paste -d, a b; join; column -t; nl; rev; tac; shuf; seq 1 10; xargs -n1 -P4 -I{} cmd {}
tee out.log; tee -a; cmd 2>&1 | tee log; cmd > /dev/null 2>&1
jq '.a.b[] | select(.x=="y") | .id' f.json; jq -r; jq -c; jq 'keys'; jq 'length'; yq '.spec.replicas' f.yaml
base64 -w0 f; base64 -d; md5sum; sha256sum -c sums.txt; xxd; od -c; strings bin | grep x
envsubst < template > out; printf '%s\n' "${arr[@]}"; date -u +%FT%TZ; date -d @1700000000; date -d '2 days ago'
```

## Disk, filesystems, LVM

```bash
lsblk -f; blkid; fdisk -l; parted /dev/nvme1n1 print
mkfs.xfs /dev/nvme1n1; mkfs.ext4 -L data /dev/xvdf
mount /dev/nvme1n1 /data; umount /data; mount -a; findmnt; cat /etc/fstab   # use UUID= in fstab, add nofail for EBS
# /etc/fstab: UUID=xxxx /data xfs defaults,nofail 0 2
xfs_growfs /data; resize2fs /dev/xvdf           # after EBS modify-volume
growpart /dev/nvme0n1 1                          # grow partition first (root volume)
pvcreate /dev/xvdg; vgcreate vg0 /dev/xvdg; lvcreate -L 20G -n lv0 vg0; lvextend -r -L +10G /dev/vg0/lv0; pvs vgs lvs
swapon --show; fallocate -l 2G /swap; mkswap /swap; swapon /swap
fsck -y /dev/xvdf (unmounted); xfs_repair
iotop; iostat -xz 1; ionice
```

## Kernel, boot, misc

```bash
uname -a; cat /etc/os-release; hostnamectl; timedatectl; localectl
sysctl -a | grep net.ipv4.ip_forward; sysctl -w net.core.somaxconn=4096; /etc/sysctl.d/99-custom.conf; sysctl -p
lsmod; modprobe br_netfilter; cat /proc/sys/...
cat /proc/meminfo /proc/cpuinfo; dmidecode -t system
chronyc sources; ntpq -p
crontab -l; crontab -e; /etc/cron.d/; cat /var/log/cron       # m h dom mon dow cmd ; @reboot ; use flock
echo $?; set -x; bash -x script.sh; shellcheck script.sh
history | tail; !!; !$; ctrl-r
alias ll='ls -lah'; export PATH=$PATH:/opt/bin; source ~/.bashrc; env; printenv X; unset X
cloud-init status; cat /var/log/cloud-init-output.log; cloud-init clean   # EC2 user data debugging
curl -s http://169.254.169.254/latest/meta-data/instance-id -H "X-aws-ec2-metadata-token: $(curl -sX PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')"   # IMDSv2
```

## Logs worth knowing

| Path | What |
|---|---|
| `/var/log/messages` (RHEL) / `/var/log/syslog` (Debian) | general |
| `/var/log/secure` / `/var/log/auth.log` | ssh, sudo |
| `/var/log/cloud-init-output.log` | EC2 user data |
| `/var/log/dmesg`, `dmesg -T` | kernel, OOM |
| `/var/log/nginx/*.log`, `/var/log/httpd/` | web |
| `/var/log/audit/audit.log` | auditd / SELinux denials (`ausearch -m avc -ts recent`) |
| `/var/log/amazon/ssm/`, `/opt/aws/amazon-cloudwatch-agent/logs/` | agents |
| `logrotate -d /etc/logrotate.conf` | test rotation |

## Performance triage order (USE method)

1. `uptime` load vs `nproc`
2. `dmesg -T | tail` errors/OOM
3. `vmstat 1` r (runnable) / b (blocked) / si so (swapping) / us sy id wa
4. `mpstat -P ALL 1` single hot core?
5. `pidstat 1` which process
6. `iostat -xz 1` await / %util
7. `free -h` memory + cache
8. `sar -n DEV 1` / `ss -s` network
9. `top` / `htop` confirm
