---
title: "Granular privileges on Linux with setpriv"
date: 2022-05-26T22:42:33-03:00
draft: false
---

`setpriv` is a neat little known tool included in many distributions' `util-linux` package. It changes privileges in a very granular way (including choosing specific Linux capabilities) and directly executes the program with changed permissions. It's basically Linux-optimized version of runit's `chpst -u`, using one fewer PID than `su`/`sudo`/`runuser`.


# Usage

`setpriv` has many options, and an exhaustive tutorial would go on for far longer. This is only a fraction of what it can do, read the `man 1 setpriv` for more details.

## Setting user and groups

This is similar to what is provided by `chpst -u`. The relevant¹ options for setting user and group in `setpriv` are:

- `--reuid` - set the user (notice this alone **won't** change groups);
- `--regid` - set the group;


Supplementary groups, which provide additional permissions (think, for example, your desktop Linux account being on `video` and `audio` groups besides the group named after your account), must be explicitly handled through one of these options:

- `--init-groups`: use the user's supplementary groups as specified in `/etc/groups`
- `--clear-groups`: deletes supplementary groups, leaving only the one specified in `--regid`.
- `--groups`: allows you manually set additional groups. Comma-separated list of group names.

This might make `setpriv` seem verbose, but provides versatility.

To reset the environment and set `HOME` and `SHELL` variables, you can use the `--reset-env` option.

For a very simple example, this `setpriv` invocation reloads newsboat RSS feeds, and could be used at a system cronjob:

```sh
setpriv --reuid user --regid user --init-groups --reset-env -- newsboat -x reload
```

For a more "real-world" example, a runit service using the built-in chpst, vs one using setpriv:

```sh
# With runit
exec chpst -u http:http myserver /server
# With setpriv - --clear-groups emulates runit's chpst
exec setpriv --reuid http --regid http --clear-groups -- myserver /server
```

As it directly executes the process after dropping privileges, `setpriv` and the program it spawns share the same PID and process monitoring tools (even the simplest and most portable ones like `runit`) can send signals (including SIGKILL) and reliably know if they're up.

¹While there are individual `--ruid --rgid --euid --egid` options, the two above are suitable for most scripts.

## Setting capabilities

However, the killer feature of setpriv is using it to add capability support without changing the file itself. Let's say we want the previous `myserver` example to be able to bind to root-only ports even though it starts a non-root user. This can be accomplished with the `net_bind_service` capability, which can be added to process by setting `--ambient-caps` and `--inh-caps` to `+net_bind_service`.

```sh
setpriv \
	--reuid http --regid http --clear-groups \
	--ambient-caps +net_bind_service --inh-caps +net_bind_service -- \
	myserver /server --port=80
```

To add more capabilites, you can separate the additions with commas:

```sh
setpriv \
	--reuid http --regid http --clear-groups \
	--ambient-caps +net_bind_service,+sys_admin --inh-caps +net_bind_service,+sys_admin -- \
	myserver /server --port=80
```

If you find yourself wanting to remove a capability, just use `-some_capability` in the value for these options. As a shorthand, you can grant/remove all capabilities with `+all` / `-all`. For example, to explicitly remove all capabilities except for `net_bind_service`, you'd write `-all,+net_bind_service` on both options.

With this change, `myserver` will have the capability even if the file itself doesn't have it.

For a more technical description of how Linux capabilities work in our case, read *Transformation of capabilities during execve()* from `man 7 capabilities`.

As I've remarked on the [original version of this post on the Artix Linux forum](https://forum.artixlinux.org/index.php/topic,3360.0.html), this can be used to help port systemd services that rely on its native support for capabilities.


# Misc tidbits

## What exactly is the problem of `su`, `sudo` and `runuser` in service scripts and cronjobs?

`setpriv` is explicitly intended to drop privileges and directly executes the program after performing the requested changes, nothing more, nothing less.

The other three perform a lot of not-really-needed setup (controlled by their files in `/etc/pam.d`) and will spawn the program as an additional process, which is less efficient on a cronjob, and unacceptable for process monitors, which need direct access to the process to work as intended. [Read more here](https://jdebp.uk/FGA/dont-abuse-su-for-dropping-privileges.html).

## Using the (non-POSIX) shell's brace expansion

In order to make scripts shorter, you can exploit the fact that the `--reuid` and `--regid` pair, as well as the `--ambient-caps` and `--inh-caps` pair, often take the same value.

```sh
# bash/*ksh/zsh
setpriv --re{u,g}id=http --init-groups --{ambient,inh}-caps=+net_bind_service -- myserver /server --port=80
# execline (included with s6-rc)
# exploits how execline handles arrays within arguments
multisubstitute
{
	define -s u,g "u g"
	define -s a,i "ambient inh"
}
setpriv --re${u,g}id=http --init-groups --${a,i}-caps=+net_bind_service --
myserver /server --port=80
```
