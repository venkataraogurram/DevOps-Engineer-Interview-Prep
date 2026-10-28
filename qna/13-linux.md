# 13. Linux

[← Back to index](README.md)

**Q101. A server is slow. Walk me through your first five minutes.**
`uptime` for load average against core count; `dmesg -T | tail` for kernel errors or OOM kills; `vmstat 1` to see whether it's CPU (us/sy high), memory (si/so swapping), or I/O (wa high, b column); `top` or `pidstat` to find the process; `iostat -xz 1` for disk await and utilisation; `ss -s` and `sar -n DEV` for network saturation; `df -h` and `df -i` because a full disk or exhausted inodes make everything slow. Then `journalctl -p err -b` for the application side. That's the USE method: utilisation, saturation, errors, per resource.

---

**Q102. Disk is 100% full but `du` doesn't add up. Why?**
A process still holds a deleted file open, so the space isn't released. `lsof | grep deleted` finds it; fix by truncating via `/proc/PID/fd/N` or restarting the process. Other causes: files under a mount point hidden by the mount, reserved blocks (5% on ext4), or inode exhaustion (`df -i`) from millions of tiny files, often session or cache directories.

---

**Q103. Explain file permissions, including setuid, setgid and sticky bit.**
Three triplets for owner, group, others: read 4, write 2, execute 1. On directories, execute means traverse. Setuid (4000) runs an executable as the file owner, like `passwd`. Setgid (2000) on a directory makes new files inherit the directory's group, useful for shared team folders. Sticky bit (1000) on a directory means only the file owner can delete their files, as in `/tmp`. ACLs with `setfacl` extend beyond owner/group/other.

---

**Q104. What happens when you run a command, from shell to process?**
The shell parses the line, expands variables and globs, resolves the command via aliases, functions, builtins, then `$PATH`. It `fork()`s a child, sets up redirections and pipes, then `exec()`s the binary. The kernel loads the ELF, the dynamic linker resolves shared libraries, and `main` runs. The parent `wait()`s and gets the exit code in `$?`. Pipes run each stage in its own process concurrently.

---

**Q105. How does systemd differ from init scripts, and how do you write a service?**
systemd manages units with dependency ordering and parallel start, restarts on failure, cgroups for resource control, and journald logging. A service unit has `[Unit]` with `After`/`Wants`, `[Service]` with `ExecStart`, `User`, `Restart=on-failure`, `EnvironmentFile`, limits, and `[Install]` with `WantedBy=multi-user.target`. `systemctl daemon-reload` after editing, `enable --now` to start on boot, `journalctl -u name -f` for logs, `systemctl edit` for drop-in overrides instead of editing vendor files.

---

**Q106. What is the OOM killer and how do you find what it killed?**
When the kernel can't reclaim memory it kills the process with the highest `oom_score`, usually the largest RSS. `dmesg -T | grep -i -E 'killed process|out of memory'` or `journalctl -k` shows the victim and the memory state. Prevent with memory limits per service (cgroups, container limits), swap as a buffer, fixing leaks, or adjusting `oom_score_adj` for critical daemons. In Kubernetes it surfaces as exit code 137 and `OOMKilled`.

---

**Q107. How do you find which process is listening on a port, and what's using a file?**
`ss -tulpn | grep :8080` (needs root for the process name) or `lsof -i :8080`. For a file or directory, `lsof /path` or `fuser -v /path`. For which process is making outbound connections, `ss -tnp` and filter by destination.

---

**Q108. Difference between hard and soft links?**
A hard link is another directory entry pointing at the same inode; the file lives until the last link is removed; can't cross filesystems or link directories. A soft (symbolic) link is a separate file containing a path; it can dangle, cross filesystems, and point to directories. `ls -li` shows inode numbers and link counts.

---

**Q109. How do you troubleshoot a service that fails to start on boot but starts manually?**
Usually ordering or environment. `journalctl -u svc -b` for the boot-time error. Common causes: it starts before the network is up (`After=network-online.target` and `Wants=network-online.target`), a mount isn't ready (`RequiresMountsFor=`), an environment variable set only in a user's shell, a dependency on DNS that isn't ready, or a file permission issue because it runs as a different user at boot. `systemd-analyze critical-chain svc` shows the ordering.

---

**Q110. What's in `/proc` and how do you use it?**
A virtual filesystem exposing kernel and process state. `/proc/PID/cmdline`, `environ`, `status`, `fd/`, `limits`, `cwd` for a process; `/proc/meminfo`, `/proc/cpuinfo`, `/proc/loadavg`, `/proc/net/tcp`; `/proc/sys/` mirrors `sysctl` tunables, for example `/proc/sys/net/ipv4/ip_forward`. Handy for reading a running process's environment or confirming its actual open files without extra tools.

---

**Q111. Explain how DNS resolution works on a Linux host.**
`/etc/nsswitch.conf` sets the order, typically files then dns. `/etc/hosts` first, then the resolvers in `/etc/resolv.conf`, which on modern systems is managed by systemd-resolved (stub at 127.0.0.53) or NetworkManager. On EC2 the VPC resolver is at the VPC base +2 (`169.254.169.253` also works); Route 53 private hosted zones and Resolver rules apply there. `dig` bypasses nsswitch and talks to the resolver directly, so if `dig` works and the app doesn't, check `/etc/hosts`, nsswitch, or the app's own resolver and caching. `resolvectl status` shows per-interface DNS.

---

**Q112. How do you harden a Linux server for a bank?**
Baseline it from a CIS benchmark via Packer: minimal packages, SSH with key-only auth and no root login, or no SSH at all via SSM; `auditd` with rules shipped to a central store; SELinux enforcing; firewall default deny; automatic security updates or immutable rebuilds; file integrity monitoring (AIDE, Wazuh); `sysctl` hardening (no IP forwarding, SYN cookies, rp_filter); separate partitions with `noexec` on `/tmp`; password policy and account lockout via PAM; time sync with chrony; centralised logs; an EDR agent. Then scan with OpenSCAP or Inspector and treat findings as pipeline failures.

---

**Q113. How do cgroups and namespaces relate to containers?**
Namespaces isolate what a process can see: PID, network, mount, UTS (hostname), IPC, user. Cgroups limit what it can use: CPU, memory, I/O, PIDs. A container is a process with its own namespaces, a cgroup for limits, a root filesystem from image layers (overlayfs), and dropped capabilities plus seccomp for security. That's why containers start fast and share the host kernel; a VM has its own kernel.

---

**Q114. Log rotation: how does it work and what breaks?**
`logrotate` runs from cron or a systemd timer with configs in `/etc/logrotate.d/`: rotate by size or time, keep N, compress, then signal the app (`postrotate` with `kill -USR1` or `copytruncate` if the app can't reopen). Breaks when the app keeps the old file descriptor open and writes to the rotated file (disk fills with "deleted" files), or when `copytruncate` loses lines written between copy and truncate. journald handles its own rotation (`SystemMaxUse`). Containers should log to stdout and let the runtime or Fluent Bit handle it.

[← Back to index](README.md)
