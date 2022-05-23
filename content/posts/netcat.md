---
title: "File Transfers with Netcat"
date: 2022-02-11T13:58:38-03:00
draft: false
---

If you have two computers on the same local network, in addition to "standard" ways of transferring data (cloud storage, Bluetooth, scp/sshfs, etc.), I'd like to add one more: the netcat (`nc`) command.

There's no need for key or PIN exchange, special hardware or working Internet connection; the only requirement is that both computers be connected to the same access point (like a router or phone hotspot).

# Installing

On many distributions, there are at least two options for netcat: GNU netcat and BSD netcat. For instance, on Arch-like distros:

```plain
# pacman -S netcat
:: There are 2 providers available for netcat:
:: Repository world
   1) gnu-netcat
:: Repository galaxy
   2) openbsd-netcat
```

Check your distro's repository for more information.

Both work for this example, and can transfer between one another. I've even done netcat transfers between BSD netcat (main computer) and GNU netcat (Termux).

# Single file transfers

## Setting up the receiving machine

To initiate a transfer of a single file, run this command on the receiving end:

```sh
nc -l 7777 > outfile
```

This will start netcat in **l**isten mode on port 7777, and will direct its output (the received data) to a file named `outfile`.

- 7777 is just an example and you can pick any number greater than 1024 as the port (numbers smaller than 1024 are reserved for system services running as root).
- Don't forget to specify the `> outfile`; otherwise, the file's contents (which might contain garbled/control characters) will be printed to the terminal. You don't want that to happen on the Linux tty.

## Finding out the receiving end's local IP address

In order to send the file, knowing the receiving end's local IP address is needed. This can be done by looking for lines with `inet` in `ip addr show`'s output and picking the first without `127.0.0.1` or `::1` (which only work within the computer).

```plain
$ ip addr show | grep inet
    inet 127.0.0.1/8  [...] # does not work on other machines.
    inet6 ::1/128     [...] # does not work on other machines.
    inet 192.168.3.22/24 [...] # bingo!
[...]
```

Pick the dot-separated four number sequence after `inet`, discard what comes after the slash, and there's your box's IP address on the local network (on the example above, `192.168.3.22`).

On some graphical environments, such as KDE, this information is given to you on the Network Settings menu ("IPv4 address").

## Sending the file

Now, go on the computer with the file you want to send, and type:

```sh
nc 192.168.3.22 7777 < /path/to/file
```

If you used a number other than 7777 on step 1, or found a different IP address on step 2, adjust the command accordingly.

The transfer is initiated, and netcat will use whatever file you specified with `<` as the standard input, sending it over the network. No indication of progress is given except for the file growing at the receiving end, however.

# Multiple files transfer

`nc` can acutally transfer _any_ kind of data over the local network -- you could leverage it to make different parts of a pipeline run on different machines.

Therefore, by creating a tarball in one machine, sending it over the network, and extracting it on another, it'd be equivalent to transferring multiple files.

```sh
# On the receiving end.
# Extracts the tarball and lists created files.
nc -l 7777 | tar xvf-
# On the sending end.
# Create a tarball and send it over a pipe to nc.
tar cf- picture1.jpg picture2.jpg | nc 192.168.3.22 7777
```

This is the logical combination of `nc` sending data over the network, and `tar` being able to bundle files.

- Tip: You can use GNU `tar`'s `--xform` option to sanitize filenames if you're sending them to an Android phone or an NTFS filesystem. For instance: `--xform='s@[^A-Za-z0-9_.-/]@@g'` to only allow normal ASCII characters in filenames.

# Progress indicators

If you're sending something big and progress indicators give you peace of mind, you can use `dd status=progress` or the dedicated [`pv`](http://www.ivarch.com/programs/pv.shtml) program:

```sh
# pv
# Receiving end
nc -l 7777 | pv > outfile
# Sending end
pv < infile | nc 192.168.3.22 7777
# dd
# Receiving end
nc -l 7777 | dd status=progress > outfile
# Sending end
dd status=progress < infile | nc 192.168.3.22 7777
```

Using `dd`/`pv` on both ends is not necessary. They're only there to measure the transfer speed, and skipping them on one of the ends is not going to prevent `nc` for establishing a connection and transferring data, as long as you keep `< >` in the right places.

You could go nuts and combine both with `tar` (with `tar | pv | nc`; `nc | pv | tar`)!

# Closing words

I hope this works as an introduction to netcat and its versatility. I usually send SSH keys through netcat as it's a very simple method requiring litte setup.

In fact, I've only shown a fraction of its power, and with creativity and CLI knowledge, many more uses can be found for it, enriching normal terminal programs with network capabilities.
