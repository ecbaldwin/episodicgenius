+++
date = "2018-05-25T17:10:00-06:00"
title = "DO Droplets and Root"
Tags = ["DigitalOcean", "DO", "ansible", "security", "root"]
+++

I've used DigitalOcean (DO) before. I think I spun up my first tiny droplet a
few years ago and I ran this blog on one for quite a while. However, it had been
a while and I never got into it enough to have needed much automation. Now, I'm
a DO employee and I will be using it frequently.

The problem is that DO droplets come up with access only to the root user. I
don't think it is right. For one thing, the root user is *the* most common user
out there. Every UNIX system has one so it is a target for attackers. Maybe it
isn't that much worse than the other common default -- that is to begin with
another relatively common user like `ubuntu` and give that user passwordless
sudo access -- but I believe it is a better starting point and can easily be
secured further. DO has so many other great things going for it, I'm willing to
let this one slide ... for now.

Ignoring security concerns, the immediate problem for me was that all the
ansible playbooks I've accumulated begin with the assumption that it will login
as an unprivileged user first and then use privilege escalation when necessary.
My normal flows were broken.

Enter my new `do-bootstrap.yaml` playbook:

{{< highlight yaml >}}
# Digital ocean starts images out with root access. This bootstrap playbook
# adds a user and sets it up for whatever else.
- hosts: all
  vars:
    username: carl
  tasks:
  - name: Add group
    group:
      name: "{{ username }}"
      state: present
  - name: Add user
    user:
      name: "{{ username }}"
      group: "{{ username }}"
      groups: sudo
      shell: /bin/bash
      append: yes
  - name: get key from file
    local_action:
      module: shell
      _raw_params: head -n 1 ~/.ssh/authorized_keys
    register: ssh_key_action
  - name: install ssh key
    authorized_key:
      user: "{{ username }}"
      key: "{{ ssh_key_action.stdout }}"
  - name: enable passwordless sudo
    lineinfile:
      path: /etc/sudoers
      state: present
      regexp: '^%sudo'
      line: '%sudo ALL=(ALL) NOPASSWD: ALL'
      validate: 'visudo -cf %s'
  - name: Disable root login
    lineinfile:
      dest: /etc/ssh/sshd_config
      regexp: "^PermitRootLogin "
      line: "PermitRootLogin no"
    notify:
      - restart sshd
  - name: Disable password login
    lineinfile:
      dest: /etc/ssh/sshd_config
      regexp: "^PasswordAuthentication "
      line: "PasswordAuthentication no"
    notify:
      - restart sshd

  handlers:
    - name: restart sshd
      systemd:
        name: sshd
        state: restarted
        daemon_reload: yes
{{< /highlight >}}

It is now the first thing that I run after booting a new droplet on DO. It gets
me to the point where many other cloud images start and fits well into my
automation.

One nice advantage to this flow is that I get to pick my own username. I could
always do that with any other image, but I never bothered. This forced me into
it and I am choosing to count that as a positive outcome.
