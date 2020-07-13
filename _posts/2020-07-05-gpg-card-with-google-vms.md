---
title: SSH into Google Compute Engine VMs with OpenPGP USB token/SmartCard
tags:
- Google Cloud Platform
image: /images/2020-07-05-gpg-card-with-google-vms-illustration.png
reddit: /r/googlecloud/comments/hqk4n4/ssh_into_google_compute_engine_vms_with_openpgp/
hn: /item?id=23823480
---

![illustration](/images/2020-07-05-gpg-card-with-google-vms-illustration.png)
Do you frequently SSH into Google Compute Engine VMs using `gcloud compute ssh ...`, and own a
physical OpenPGP-compatible SmartCard/USB key? This post, also serving as a future note to myself,
shows how to use these together for ~~fun and profit~~ convenience and security.

## Prerequisites

- Linux (Mac and maybe even Windows may use a similar trick, but I have no way to test)
- Physical OpenPGP-compatible USB key or a SmartCard at hand.
  - For example [YubiKey](https://support.yubico.com/support/solutions/articles/15000006420-using-your-yubikey-with-openpgp),
    [NitroKey](https://www.nitrokey.com/documentation/openpgp-create-backup), or a
    [SmartCard](https://en.wikipedia.org/wiki/OpenPGP_card) plus a reader.
- The USB key/SmartCard configured (also) for SSH authentication (per links above).
  - ```bash
    $ gpg --card-status  # output should contain something similar to
    Authentication key: 0EFC F3FD 3AFC 6D57 E0A4  1FB9 0C44 AEC0 3034 55E0
    ```
- [Configured gpg-agent to act as an SSH agent](https://www.bootc.net/archives/2013/06/09/my-perfect-gnupg-ssh-agent-setup/):
  - ```bash
    $ ssh-add -l  # should say something like
    2048 SHA256:cDS0g8TiH0bJJthhJFW/GqKDmCnopkcEKSiRuEEAXmY cardno:000000012345 (RSA)
    ```

## TL;DR:

```bash
$ ssh-add -L > ~/.ssh/google_compute_engine.pub  # overwrite (!) the public key with OpenPGP one
$ echo "fake content, actual private key on OpenPGP HW token" > ~/.ssh/google_compute_engine
```

## How Did We Get There

`gcloud compute ssh` is just a tiny wrapper around `ssh` that does a couple of things on top of it:
1. uses `~/.ssh/google_compute_engine(.pub)` SSH (public) key file path instead of standard
   `~/.ssh/id_rsa(.pub)`,
2. generates the key-pair automatically if it doesn't exist,
3. updates target VM metadata (its `~/.ssh/authorized_keys`) with the public SSH key before
   connecting (the convenient, and tricky, part).

It should be possible to fool `gcloud compute ssh` by supplying the OpenPGP-backed public SSH key
at the location it expects it, let's try:

```bash
$ ssh-add -L > ~/.ssh/google_compute_engine.pub
$ gcloud compute ssh bencher
WARNING: The private SSH key file for gcloud does not exist.
WARNING: Your SSH key files are broken.
private key (NOT FOUND) [/home/strohel/.ssh/google_compute_engine]
public key  (OK)        [/home/strohel/.ssh/google_compute_engine.pub]
We are going to overwrite all above files.
Do you want to continue (y/N)?  N
```

Hmmm, `gcloud` was being a bit too smart here. We don't have the private key in file, because that's
the entire point of confining it into a separate hardware device. Let's try to cheat a bit more:

```bash
$ touch ~/.ssh/google_compute_engine
$ gcloud compute ssh bencher
WARNING: The private SSH key file for gcloud is empty.
(...)
```

Still no avail. Let's outsmart `gcloud` completely:

```bash
$ echo "fake content, actual private key on OpenPGP HW token" > ~/.ssh/google_compute_engine
$ gcloud compute ssh bencher
(...)
Last login: ...
```

Yay. That's it, we need to write the public key in SSH format into `google_compute_engine.pub`, and
some random content into `google_compute_engine`.

## Notes

1. I have tested this successfully with [YubiKey 4](https://support.yubico.com/support/solutions/articles/15000006486-yubikey-4)
   and [ZeitControl "BasicCard" SmartCard (no longer) provided by FSFE](https://fsfe.org/news/2017/news-20171116-01.en.html).
1. If you try to SSH without your OpenPGP token available, the error will look like:
   ```bash
   $ gcloud compute ssh bencher
   Load key "/home/strohel/.ssh/google_compute_engine": invalid format
   (...)
   ```
1. [Some folks say they need to write the authentication keygrip into `~/.gnupg/sshcontrol` file.](
   https://stackoverflow.com/a/48922829/4345715) For me this works without touching the `sshcontrol`
   file (it contains only comments). I suspect that it is used only for file-based GPG-as-SSH keys,
   and not relevant for GPG-as-SSH keys backed by HW tokens.
1. To resolve some problems with card "PIN Entry" dialog not showing up sometimes, I have this
   snippet in my `~/.bashrc`:
   ```bash
   # Needed to make gpg-agent as ssh-agent pop-up pinentry correctly
   # https://unix.stackexchange.com/a/218401
   echo UPDATESTARTUPTTY | gpg-connect-agent >/dev/null
   ```
1. Sometimes, during SSH operations like git pushes, I get `Agent refused operation` errors from
   `gpg-agent`. A re-plug and/or restart of the PC/SC Smart Card Daemon (`systemctl restart pcscd`)
   helps.
1. For modern OpenSSH versions (8.2+ on both client and server) and security tokens (supporting
   FIDO2/U2F) a technically different alternative (that does not involve OpenPGP/GnuPG) exists, [as
   described in Google Cloud docs](https://cloud.google.com/compute/docs/tutorials/ssh-with-sk) and
   [elsewhere](https://www.stavros.io/posts/u2f-fido2-with-ssh/).
