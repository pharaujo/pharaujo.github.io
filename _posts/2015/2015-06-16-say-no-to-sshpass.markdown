---
layout: post
title:  "Say 'No' to sshpass"
categories: ssh sshpass ansible
---

_December 3, 2019: updated post to work on macOS 10.14_

I recently started fiddling with Ansible. I'm in a position where I see what all the fuss is about, but its quirks still nag me; one of which is the requirement to use `sshpass` for when you don't have your SSH keys in place.

I *really* don't like `sshpass` - mostly because of security concerns - but the end goal of SSH automation is still worth pursuing, I think.

So, this exercise started by being stubborn in believing you could mostly do what `sshpass` does with plain vanilla `ssh`! In fact, I would argue that this might be A Better Way™, but you're free to disagree. =)

I want to store my passwords in an OS X keychain and have them read straight to `ssh`, so first we'll create a secure keychain for this purpose:

```bash
# create a new keychain
$ security create-keychain -P test.keychain
# have it lock on sleep or after 5min
$ security set-keychain-settings -lu -t 300 test.keychain
```

OS X command line tools for system management seem to be an after-thought, as I accidentally messed up my keychain search index while researching for this post and could only recover by using the GUI `Keychain Access` app (that might be a story for a later post).

In any case, securely adding a new password to the keychain doesn't seem possible through the CLI as the tool mandates inserting the password as a command line argument. The hack I came up with for not having the password stored in the clear on the shell history file was to write it elsewhere (anywhere you can type text), copying to the clipboard (I know, I know -\_- ) and having the shell read from the clipboard:

```bash
$ security add-generic-password -a <username> -s ldap -w $(pbpaste) \
           test.keychain
```

The other way you could go about it would be to use the `Keychain Access` app and create the password item there.

Having done that, we now have to coerce `ssh` to use the password in the keychain. This was a lot harder than I previously thought, as `ssh` tries **very** hard to force you to type the password interactively, for security reasons. Having said that, here is what the `man` page states:


    SSH_ASKPASS           If ssh needs a passphrase, it will read the passphrase
                          from the current terminal if it was run from a terminal.
                          If ssh does not have a terminal associated with it but
                          DISPLAY and SSH_ASKPASS are set, it will execute the
                          program specified by SSH_ASKPASS and open an X11 window
                          to read the passphrase. (...)


This means we can use `SSH_ASKPASS` environment variable to pipe a password into `ssh`, as long as:

 -  `ssh` does not have a terminal associated with it;
 -  there is a `DISPLAY` environment variable set.

This was made so that X11 password prompts could be used with `ssh`. As we're on OS X, this is kind of irrelevant.

Oh well.

The hard part here is tricking `ssh` to run without an associated terminal and, after several failed attempts, I had to resort to The Internets. Luckily, I'm not the first person to have had this idea so [sample code was readily available](http://silmor.de/notty.php). In fact, the linked post has almost everything you need to do `ssh` automation; it just needed a little OS X love to work. I've set up a [github repository](https://github.com/pharaujo/notty) with the code so that this is easily reproduceable. You just need to clone the repo and follow the instructions to install `notty`.


The fun part is that `SSH_ASKPASS` just needs to point to an executable that outputs the password to stdout. Of course that would be lame and terribly insecure, so we just need to write a script that, with your permission, grabs your password from the keychain.

Place the following in `~/bin/askpass`:

```bash
#!/usr/bin/env bash
/usr/bin/security find-generic-password -a <username> -s ldap -w \
                  test.keychain
```

and make it executable:

```bash
$ chmod u+x ~/bin/askpass
```

The parameters you use here are the same you used when creating your generic-password item earlier.

We now have the foundations to passwordless (sort of) `ssh` and can try it with a server with password authentication:

```bash
$ DISPLAY=:99 SSH_ASKPASS="~/bin/askpass" notty ssh -q <server> uptime
```

If everything went well, a keychain prompt should appear asking for the keychain password. After that you'll feel the sweet bliss of realizing you just had to type a password so that you don't need to type another. ^\_^'

Most of the pieces are now in place to replace `sshpass` (that was the point of the exercise, remember?). As `ansible` is hard-coded to require `sshpass` for password-based authentication and to [disallow password authentication when not using -k](https://github.com/ansible/ansible/blob/ce3ef7f4c16e47d5a0b5600e1c56c177b7c93f0d/lib/ansible/plugins/connections/ssh.py#L107), we need to fool it into using our `SSH_ASKPASS` setup.

Our cool `sshpass` replacement (place it `/usr/local/bin/sshpass`):

```bash
#!/usr/bin/env bash

export DISPLAY=:99
export SSH_ASKPASS="$HOME/bin/askpass"

[[ $1 == -d* ]] && shift
notty $@
```

`ansible` uses the `-d` flag to tell `sshpass` which file descriptor to read the password from. As we don't care about that, we just ignore it and use the rest of the generated command directly.

```bash
$ cat /tmp/a
server0[1:3]

$ ansible -i /tmp/a all -m ping -k
SSH password:
server01 | success >> {
    "changed": false,
    "ping": "pong"
}

server02 | success >> {
    "changed": false,
    "ping": "pong"
}

server03 | success >> {
    "changed": false,
    "ping": "pong"
}
```

Success!

This, of course, is not ideal as `ansible` prompts you for a password anyway and then our replacement `sshpass` disregards that entirely. Fixing this requires patching `lib/ansible/plugins/connections/ssh.py`, which is a lot uglier than to type gibberish on the `ansible` prompt.

And that concludes our exercise for now :)

Thanks to [@kintoandar](https://twitter.com/kintoandar) for all the help with Ansible, and for pushing me to write this post!
