# Adding an SSH Public Key to a Vagrant VM

Vagrant gives every box its own auto-generated insecure/replacement keypair and the handy `vagrant ssh` shortcut. But sometimes you need *your own* public key inside the box — most commonly so another tool can reach it over plain SSH without `vagrant ssh`. The classic case is Ansible: the playbook can't connect to the VM until your public key is in its `authorized_keys`.

This article covers four ways to get a public key into a Vagrant VM, when to use each, and how to verify and troubleshoot the result. Based on and expanded from [jhooq's guide](https://jhooq.com/vagrant-copy-public-key).

## First: Generate a Key (If You Don't Have One)

On your host, create a keypair if you don't already have one. Ed25519 is the modern default; RSA still works everywhere.

```bash
# Modern, recommended
ssh-keygen -t ed25519 -C "you@host"

# RSA alternative (use 4096 bits)
ssh-keygen -t rsa -b 4096 -C "you@host"
```

Accept the default location (`~/.ssh/id_ed25519` or `~/.ssh/id_rsa`) and set a passphrase if you want one. The public key is the `.pub` file — that's what goes into the VM. Never copy the private key into the box.

## Method 1 — `ssh-copy-id` (Simplest, After Boot)

Once the box is up and reachable on a known IP, `ssh-copy-id` appends your public key to the VM's `authorized_keys` in one command.

```bash
# vagrant is the default user on Vagrant boxes
ssh-copy-id -i ~/.ssh/id_ed25519.pub vagrant@192.168.56.10
```

Notes:
- Use the `vagrant` user — it's the default account on virtually all base boxes.
- The IP is your VM's reachable address. Give the box a private network in the Vagrantfile so it has a stable IP:

  ```ruby
  config.vm.network "private_network", ip: "192.168.56.10"
  ```
- This needs SSH to already work. Vagrant boxes accept the built-in key, so `ssh-copy-id` may prompt for the `vagrant` password (`vagrant`) if you aren't using that key, or you can point it at Vagrant's private key with `-o IdentityFile=...`.

If you don't want a private network, you can reach the box over the **forwarded port** Vagrant sets up by default — localhost's `2222` maps to the box's `22`:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 vagrant@localhost
```

The default credentials are user `vagrant` / password `vagrant`, so when prompted you can just enter `vagrant`. Afterward you can log in without a password:

```bash
ssh -i ~/.ssh/id_ed25519 -p 2222 vagrant@localhost
```

(If `2222` is taken, Vagrant auto-increments the forwarded port — check `vagrant ssh-config` for the actual `Port`.)

Best when: the box is already running and you just want to push a key interactively.

## Method 1b — Manual Pipe Over SSH (No `ssh-copy-id`)

If `ssh-copy-id` isn't available (it's missing on some minimal or Windows setups), you can do the same thing with a plain pipe — read the public key on the host and append it to `authorized_keys` inside the box in one command:

```bash
cat ~/.ssh/id_ed25519.pub | \
  ssh -p 2222 vagrant@localhost 'cat >> ~/.ssh/authorized_keys && echo "SSH key copied."'
```

This is exactly what `ssh-copy-id` does under the hood, minus the duplicate-detection. It's handy as a portable fallback, but `ssh-copy-id` is preferable when present because it skips keys that are already installed.

Best when: `ssh-copy-id` isn't installed and you want a one-liner.

## Method 2 — Inline Shell Provisioner (Ruby `File` Read)

Because the `Vagrantfile` is Ruby, you can read your public key at provision time and append it inside the box. This bakes the key in automatically on `vagrant up` / `vagrant provision`.

```ruby
config.vm.provision "shell" do |s|
  ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_ed25519.pub").first.strip
  s.inline = <<-SHELL
    mkdir -p /home/vagrant/.ssh /root/.ssh
    echo "#{ssh_pub_key}" >> /home/vagrant/.ssh/authorized_keys
    echo "#{ssh_pub_key}" >> /root/.ssh/authorized_keys
    chown -R vagrant:vagrant /home/vagrant/.ssh
    chmod 700 /home/vagrant/.ssh /root/.ssh
    chmod 600 /home/vagrant/.ssh/authorized_keys /root/.ssh/authorized_keys
    # Avoid duplicates on repeated provisioning:
    sort -u -o /home/vagrant/.ssh/authorized_keys /home/vagrant/.ssh/authorized_keys
  SHELL
end
```

- `#{Dir.home}` resolves the host user's home directory portably instead of hard-coding `/home/username`.
- Setting ownership and permissions matters — SSH refuses keys if `.ssh` (700) or `authorized_keys` (600) are too permissive.
- The `sort -u` line keeps re-provisioning idempotent so the key isn't appended twice.

Best when: you want the key present automatically every time the box is built, for both `vagrant` and `root`.

## Method 3 — File Provisioner

The `file` provisioner copies a file from host to guest. Note it only *copies the file* — it does not append to `authorized_keys`, so pair it with a small shell step.

```ruby
# Copy the public key into the box
config.vm.provision "file",
  source: "~/.ssh/id_ed25519.pub",
  destination: "/tmp/host_key.pub"

# Then install it
config.vm.provision "shell", inline: <<-SHELL
  mkdir -p /home/vagrant/.ssh
  cat /tmp/host_key.pub >> /home/vagrant/.ssh/authorized_keys
  chown -R vagrant:vagrant /home/vagrant/.ssh
  chmod 700 /home/vagrant/.ssh
  chmod 600 /home/vagrant/.ssh/authorized_keys
SHELL
```

A common mistake is `destination: "~/.ssh/id_rsa.pub"` alone — that just drops the file there without adding it to `authorized_keys`, so SSH still won't accept it. The file provisioner also runs as the SSH user (no sudo), so use `/tmp` for the drop and let the shell step place it.

Best when: you need the raw file in the box (not only the authorized key), or prefer separating copy from install.

## Method 4 — Use Vagrant's Own Key via `ssh-config`

This one doesn't add *your* key at all — it reuses the private key Vagrant already generated, so you can SSH in directly (useful for scripts, Ansible `ansible_ssh_private_key_file`, or IDE remote SSH) without running `vagrant ssh`.

```bash
vagrant up
vagrant ssh-config
```

Output looks like:

```
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  IdentityFile /path/to/project/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  StrictHostKeyChecking no
```

Use those values to connect directly:

```bash
ssh -i /path/to/project/.vagrant/machines/default/virtualbox/private_key \
    -o PasswordAuthentication=no \
    -p 2222 vagrant@127.0.0.1
```

Even better, write it into your SSH config so `ssh vagrant-dev` just works:

```bash
vagrant ssh-config --host vagrant-dev >> ~/.ssh/config
```

Best when: you don't need your own key in the box — you just want non-`vagrant ssh` access for tooling.

## Method 5 — Let Vagrant Manage Your Key (`config.ssh`)

The most "Vagrant-native" option is to tell Vagrant to use *your* keypair instead of the auto-generated one. Then `vagrant ssh`, `vagrant ssh-config`, and any provisioner all authenticate with your key — no manual `authorized_keys` editing.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  # Use your own private key instead of Vagrant's generated one
  config.ssh.private_key_path = "~/.ssh/id_ed25519"
  # Don't replace the key with a fresh insecure->secure swap on first boot
  config.ssh.insert_key = false
end
```

How it behaves:
- `config.ssh.private_key_path` points Vagrant at your private key on the host; the matching public key must already be in the box's `authorized_keys` (many base boxes trust the well-known insecure key, so pair this with one of the append methods the first time, or bake it into a custom box).
- `config.ssh.insert_key = false` stops Vagrant from generating and swapping in a per-machine keypair on first boot, so your key remains the one that works.
- You can also list multiple keys: `config.ssh.private_key_path = ["~/.ssh/id_ed25519", ".vagrant/.../private_key"]`.

Best when: you want your key to be *the* key Vagrant uses everywhere, not just an extra entry in `authorized_keys`.

## Method 6 — Ansible Provisioner (`authorized_key` Module)

Since the usual reason for all this is "so Ansible can connect," you can let Vagrant run Ansible and have the `authorized_key` module install the key idempotently — no shell string-appending, no duplicate entries.

```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "provision.yml"
end
```

```yaml
# provision.yml
- hosts: all
  become: true
  tasks:
    - name: Add my public key to the vagrant user
      ansible.posix.authorized_key:
        user: vagrant
        state: present
        key: "{{ lookup('file', lookup('env', 'HOME') + '/.ssh/id_ed25519.pub') }}"
```

The `authorized_key` module handles permissions, ownership, and de-duplication for you, and it's declarative — re-running the playbook is safe. Use `ansible_local` instead of `ansible` if you don't have Ansible installed on the host and want it to run inside the box.

Best when: Ansible is already in your workflow, or you want idempotent, declarative key management.

## Which Method Should I Use?

| Method | Automatic on `vagrant up`? | Adds *your* key? | Best for |
|--------|:--:|:--:|----------|
| `ssh-copy-id` | No | Yes | Quick one-off after boot |
| Inline shell (Ruby `File`) | Yes | Yes | Baking your key in for every build |
| File provisioner | Yes | Yes (with shell step) | Needing the raw file, or separating copy/install |
| `vagrant ssh-config` | N/A | No (reuses Vagrant's key) | Tooling access without `vagrant ssh` |
| `config.ssh` (native) | Yes | Yes (Vagrant uses it) | Making your key the one Vagrant uses everywhere |
| Ansible `authorized_key` | Yes | Yes | Idempotent, declarative; Ansible-based workflows |

## Verify It Worked

```bash
# From the host, using your key
ssh -i ~/.ssh/id_ed25519 vagrant@192.168.56.10 "hostname && whoami"

# Check the key landed inside the box
vagrant ssh -c "cat ~/.ssh/authorized_keys"
```

For Ansible, confirm connectivity with your key and user:

```bash
ansible -i '192.168.56.10,' all -u vagrant \
  --private-key ~/.ssh/id_ed25519 -m ping
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Permission denied (publickey)` | Key not in `authorized_keys`, or wrong user | Confirm you used `vagrant@`; re-check the append step |
| Key ignored, still asks for password | Bad permissions on `.ssh`/`authorized_keys` | `chmod 700 ~/.ssh`, `chmod 600 authorized_keys`, correct ownership |
| File provisioner "worked" but SSH fails | File copied but never appended | Add the shell step that `cat`s it into `authorized_keys` |
| Can't reach the IP | No private network / wrong IP | Add `private_network` in the Vagrantfile; check `vagrant ssh-config` |
| Duplicate keys after re-provision | Append runs every time | Use `sort -u` on `authorized_keys` (Method 2) |
| `ssh-copy-id` can't authenticate | Not using Vagrant's key/password | Pass `-o IdentityFile=<vagrant private_key>` or the `vagrant` password |

## Security Note

These techniques are meant for local development and test VMs. Only ever copy **public** keys into a box, keep the private key on your host, and don't commit a project's `.vagrant/` private key into version control. Adding a key to `root`'s `authorized_keys` (Method 2) is convenient in a lab but should be avoided on anything resembling production.

## References

- [Vagrant provisioning documentation](https://developer.hashicorp.com/vagrant/docs/provisioning) — official docs
- [Vagrant `ssh-config` command](https://developer.hashicorp.com/vagrant/docs/cli/ssh_config) — official docs
