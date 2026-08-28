# Troubleshooting Log

Real errors encountered while building this project, their root cause, and the fix. Kept in chronological/topical order rather than alphabetical, since later entries sometimes build on earlier ones.

---

## 1. `ssh-copy-id`: "No identities found"

**Symptom:**
```
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed:
/usr/bin/ssh-copy-id: ERROR: No identities found
```

**Cause:** `ssh-copy-id` looks for a key at the default path (`~/.ssh/id_ed25519` or `~/.ssh/id_rsa`) unless told otherwise. A custom-named key (e.g. `ansible_controle`) is not found automatically.

**Fix:** point explicitly to the public key file:
```bash
ssh-copy-id -i ~/.ssh/ansible_controle.pub ansible@<target-ip>
```

---

## 2. SSH key stored in `/root`, inaccessible to other sudo users

**Symptom:** an admin with full `sudo` rights still gets `Permission denied` reading the key.

**Cause:** `/root` is mode `700`, owned by `root:root`. `sudo` grants privilege escalation for commands explicitly run with it — it does not make `/root` traversable for an unprivileged shell just because the user *could* escalate.

**Fix:** move automation keys out of `/root` into a dedicated service account's home directory:
```bash
sudo useradd -m -s /bin/bash ansible
sudo mkdir -p /home/ansible/.ssh
sudo mv /root/.ssh/ansible_controle* /home/ansible/.ssh/
sudo chown -R ansible:ansible /home/ansible/.ssh
sudo chmod 700 /home/ansible/.ssh
sudo chmod 600 /home/ansible/.ssh/ansible_controle
sudo chmod 644 /home/ansible/.ssh/ansible_controle.pub
```

---

## 3. `ansible all -m ping`: "provided hosts list is empty, only localhost is available"

**Cause:** `ansible.cfg` (and therefore the inventory path it points to) is only auto-loaded when Ansible is run from the same directory it lives in. Running the command from a different working directory silently falls back to a localhost-only default.

**Fix:**
```bash
pwd                     # confirm you're in the project root
cat ansible.cfg         # confirm the inventory path is correct relative to this location
ansible -i inventory/hosts.ini all -m ping   # or run from the correct directory
```

Also check for Windows-style line endings if the file was ever touched by a non-Linux editor:
```bash
file inventory/hosts.ini   # look for "CRLF line terminators"
dos2unix inventory/hosts.ini
```

---

## 4. `ansible all -m ping` connects as `root` instead of the intended user

**Cause:** `group_vars/all.yml` (holding `ansible_user`, `ansible_ssh_private_key_file`) was placed as a sibling of `inventory/`, not **inside** it. Ansible only auto-loads `group_vars/` when it sits in the same directory as the inventory file itself.

**Fix:**
```bash
mv /home/ansible/group_vars /home/ansible/inventory/group_vars
```
Correct structure:
```
inventory/
├── hosts.ini
└── group_vars/
    └── all.yml
```

---

## 5. Template error: `'<variable>' is undefined`

**Symptom (recurring, several variations):**
```
'mgmt_ip' is undefined
'management_ip' is undefined
```

**Cause:** a single shared template file contained configuration blocks for *multiple hosts* (e.g. both `mgmt_config` and `web_config` tables) with no condition separating them. Ansible renders the entire file for whichever host it's currently processing — including sections meant for a different host — and fails when it hits a variable that was never loaded for the current host (because it was only ever defined in that other host's `group_vars` file, or wasn't relevant to it at all).

**Fix:** wrap each host-specific section in a condition based on Ansible's built-in `group_names` fact (automatic, derived purely from inventory group membership — independent of where variable *values* are stored):
```jinja2
{% if 'management' in group_names %}
table inet mgmt_config { ... }
{% endif %}

{% if 'web' in group_names %}
table inet web_config { ... }
{% endif %}
```

**Related fix — cross-host variable references:** if Host A's rules need to reference Host B's IP (e.g. Management needs Web's address for an outbound rule), and that value only lives in Host B's own `group_vars` file, it will be undefined while processing Host A. Consolidate any value referenced by more than one host into the shared `group_vars/all.yml` file instead of a single host's file.

---

## 6. `nft -f /etc/nftables.conf` silently succeeds, but the ruleset keeps duplicating on every deploy

**Cause:** an `nft` config file is not "replace everything" by default — each `table`/`chain`/`rule` block is effectively an `add` operation. Without an explicit flush, re-running the same file on top of an already-loaded ruleset appends a second copy alongside the first.

**Fix:** always start the template with:
```jinja2
flush ruleset;
```
Clean up any already-duplicated state once, manually:
```bash
sudo nft flush ruleset
```

---

## 7. `apt install <package>` fails despite allowing port 443 outbound

**Cause:** Ubuntu's default package mirrors (`archive.ubuntu.com`, `security.ubuntu.com`) use **plain HTTP (port 80)**, not HTTPS. Allowing only 443 blocks package installation entirely. Also worth checking: package mirrors resolve to multiple, load-balanced IPs — an `ip daddr`-scoped rule (fine for a single stable destination like GitHub) will intermittently fail for mirror traffic if scoped to one specific address.

**Fix:**
```bash
cat /etc/apt/sources.list   # confirm http:// vs https://
```
Allow port 80 (unscoped, or scoped to a broad range) alongside 443 — treated as a deliberate, non-standing maintenance-window rule rather than a permanent allowance.

**Why HTTP is an acceptable exception here:** package integrity is enforced independently via GPG signature verification — `apt` validates every package against Ubuntu's trusted signing key before installing, regardless of transport. HTTP exposes *which* packages are being installed to a network observer, but a tampered package in transit would still be rejected.

---

## 8. `sudo: interactive authentication is required` during an Ansible `become: yes` task

**Cause:** SSH key authentication proves *identity*; `sudo` is a separate gate that, by default, demands the account's own password before escalating to root. Ansible cannot answer an interactive password prompt.

**Fix:** grant passwordless sudo to the automation account, via a **dedicated file**, not by editing `/etc/sudoers` directly:
```bash
sudo visudo -f /etc/sudoers.d/ansible
```
Contents:
```
ansible ALL=(ALL) NOPASSWD: ALL
```

**If this still fails after creating the file**, check in order:
```bash
sudo cat /etc/sudoers.d/ansible        # confirm exact content
ls -la /etc/sudoers.d/ansible          # must be mode 0440, owned by root:root
sudo chmod 0440 /etc/sudoers.d/ansible
sudo chown root:root /etc/sudoers.d/ansible
sudo visudo -c                          # validates syntax of all sudoers files
```

**Why editing `/etc/sudoers` directly is fragile:** sudoers evaluates every matching rule across the main file and everything pulled in via `#includedir /etc/sudoers.d` (which typically sits near the end of the default file), and the **last matching rule wins**. A rule inserted into the middle of an already-populated main file has to correctly out-rank every other existing rule by position; a dedicated file in `sudoers.d/` is automatically evaluated last, guaranteeing precedence without needing to reason about the rest of the file.

---

## 9. nftables service reload fails, but `nft -f /etc/nftables.conf` succeeds manually

**Diagnosis:** check the actual service log — **on the specific host that failed**, not a different one:
```bash
sudo journalctl -xeu nftables.service
```
Live view while retrying:
```bash
sudo journalctl -fu nftables.service
```

**Note:** `Active: active (exited)` with `status=0/SUCCESS` on every listed process is the **correct, healthy** state for `nftables.service` — it is a `oneshot`-type service that runs `nft -f` once and exits; there is no long-running daemon to stay "active (running)." Only `failed` or a nonzero exit status indicates an actual problem.

---

## 10. nftables syntax error: `tcp accept` (rule with no field/action match)

**Cause:** `tcp` alone cannot precede `accept` — it needs an explicit field, most commonly `dport` (or `sport`).

**Fix:**
```
ip daddr <ip> tcp dport <port> accept
```
Not:
```
ip daddr <ip> tcp accept
```

---

## 11. SSH to a host times out despite correct source-address rule, correct `sshd` config, and correct key setup

**Symptom:** `Connection timed out` (not "refused") — a silent drop, not an active rejection.

**Cause:** the `output` chain's hook declaration was mistakenly set to `hook input` instead of `hook output`:
```
chain output {
    type filter hook input priority 0; policy drop;   # WRONG — should be hook output
    ...
}
```
A chain's *name* (`chain output { ... }`) is just a label; only the `hook` value determines when the kernel actually invokes it. With this mistake, two chains were both attached to the `input` hook (one correctly, one unintentionally), while nothing correctly governed real outbound traffic.

**Fix:** verify each chain's `hook` value matches its intended direction:
```
chain input  { type filter hook input priority 0; ... }
chain output { type filter hook output priority 0; ... }
```

**Lesson:** when a firewall symptom doesn't match the visible rules, check the chain *declaration* line itself, not just the rules inside it — it's easy to skim past since it looks like boilerplate.

---

## 12. DNS resolution intermittently fails for one specific domain, others resolve fine

**Symptom:** `nslookup github.com` → `communications error to 127.0.0.53`; other domains resolve normally.

**Diagnostic sequence:**
```bash
dig @8.8.8.8 github.com          # bypasses the local resolver stub entirely — tests the real network/firewall path
resolvectl query github.com      # more verbose than nslookup; shows which upstream server was tried and why it failed
resolvectl status                 # confirms configured DNS servers per interface
systemctl status systemd-resolved # confirms the local resolver service itself is healthy
```

**Note:** `nslookup <ip-address>` performs a **reverse** DNS lookup (PTR query) — different from a normal forward lookup, and not a meaningful way to "test" a resolver.

**Resolution in this project's case:** transient — resolved on its own without a config change, most likely a brief local resolver cache/timing issue, possibly coinciding with an nftables reload mid-query. Not fully root-caused; documented as a known transient symptom, with `dig @8.8.8.8` as the fast way to confirm whether it's a real network/firewall issue vs. a local resolver hiccup.

---

## General diagnostic habit

When something is "undefined" or "not working" in this Ansible project, check in order:
1. Which host is this happening on?
2. Which inventory group does that host actually belong to?
3. Which `group_vars` file(s) does Ansible load for that group?
4. Does the value genuinely exist in one of *those* files — or only in a file scoped to a different host?

When a *service* fails to apply a *config file*, check both layers:
1. `journalctl -xeu <service>` — what did systemd see?
2. Run the underlying command manually (e.g. `nft -f <file>`) — usually gives a more precise, line-numbered error than the service layer.
