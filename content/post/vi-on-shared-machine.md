+++
date = "2018-05-26T09:52:00-06:00"
title = "VI on Shared Machines"
Tags = ["ansible", "vi", "bash"]
+++

I've worked quite a bit on storage and networking products based on Linux
servers. One thing that I run into often is the need to share development
machines with a team of other developers. They get re-imaged quite often. Since
they come and go, we don't bother to spend much time customizing them and there
is usually just one user that we all share.

The worst about all of this for me personally is that they always come up with
bash and emacs style line editing in the shell. This drives me nuts! I'm
constantly having to type `set -o vi` so that I can navigate history and edit
commands the way I'm used to.

I've been searching for years for a nice way to make this automatic without
forcing my co-workers to adopt vi-style command line editing (because that
drives them crazy). This has been a holy grail quest for a few years.

1. I tried an elaborate scheme where I wrapped the ssh command to run the
   equivalent of `exec env SHELLOPTS=vi /usr/bin/ssh ${1+"$@"}` and then changed
   the sshd server to accept SHELLOPTS as user-supplied environment. I never
   really felt good about that.
1. I tried wrapping ssh to add `bash -o vi` to the command. I had to parse the
   entire command line and analyze it to see if I really wanted to add it or
   not. There were corner cases and I found it totally broke when I logged into
   a Cirros or Busybox system.

I'm pretty sure I tried a handful of other things and some variations on the
above themes over about 6-8 years. However, this time I think I've found
something that I'm pretty happy with as long as the remote machine uses openssh.
I like it for a couple of reasons:

1. I don't have to do anything on the client side which could break if I'm
   logging into something like Cirros or Busybox. It is server-side only.
1. It allows choosing to switch to vi-style editing per ssh key basis. This
   means my co-workers, using their own keys, get the default behavior.

The idea is to copy a small script to the remote machine which will either run
`$SSH_ORIGINAL_COMMAND` or a bash login shell with vi-style editing turned on if
it isn't set. Then, run it from `.ssh/authorized_keys`.

{{< highlight yaml >}}
---
- hosts: all
  tasks:
  - name: get key from file
    local_action:
      module: shell
      _raw_params: head -n 1 ~/.ssh/authorized_keys
    register: ssh_key_action
  - name: copy helper
    copy:
      content: |
        #!/bin/bash

        # Call from .ssh/authorized_keys with an entry like this
        #
        #     command="~/.bash-vi-login" ssh-ed25519 AAAA..... Carl Baldwin

        # If no command was given, login with bash in vi mode
        [ -z "$SSH_ORIGINAL_COMMAND" ] && exec bash -l -o vi

        eval exec "$SSH_ORIGINAL_COMMAND"
      dest: ~/.bash-vi-login
      mode: a+x
  - name: install ssh key
    authorized_key:
      user: "{{ ansible_user }}"
      key: "{{ ssh_key_action.stdout }}"
      key_options: 'command="~/.bash-vi-login"'
{{< /highlight >}}

This requires a one time step for each new machine but I can easily work that
into the bootstrap automation that brings up new machines. It is a nice
light-weight way of sharing ephemeral machines with co-workers without annoying
anyone.
