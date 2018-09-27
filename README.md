# fedora-coreos-login-messages

Runtime scripts, systemd unit files, tmpfiles, and installer scripts to provide an `issue/motd` mechanism for RHCOS/FCOS. To be distributed as an RPM, with some additional manual configuration required to work with software like PAM, agetty, ...

## Installation

Scratch build: https://koji.fedoraproject.org/koji/taskinfo?taskID=29923592

1. Load up a VM with RHCOS, FCOS, fedora/28-atomic-host, fedora/28-cloud-base...
2. SSH in and log in as root, `sudo su`
3. Do the following (replace dnf with rpm-ostree for \*COS, atomic):

```
# unlink /etc/issue /etc/motd
# rm -f /etc/issue /etc/motd
# curl --remote-name-all https://kojipkgs.fedoraproject.org//work/tasks/3592/29923592/coreos-ux-0.1-1.fc28.noarch.rpm https://kojipkgs.fedoraproject.org//work/tasks/3592/29923592/coreos-ux-issuegen-0.1-1.fc28.noarch.rpm https://kojipkgs.fedoraproject.org//work/tasks/3592/29923592/coreos-ux-motdgen-0.1-1.fc28.noarch.rpm https://kojipkgs.fedoraproject.org//work/tasks/3592/29923592/coreos-ux-profile-0.1-1.fc28.noarch.rpm
# dnf install coreos-ux-*
# systemctl reboot
```

4. SSH back in

5. Note: have to manually start the units right now (issuegen only triggers because of the udev rule)

```
systemctl start motdgen.service motdgen.path issuegen.service issuegen.path
``

## RPM build

```
$ git clone https://github.com/rfairley/fedora-coreos-login-messages && cd fedora-coreos-login-messages
$ ./rpm-build.sh
```

Then edit `config.vm.box` in the `Vagrantfile` in this repo. I named an RHCOS box `rhcos` on my system. Can also install on `fedora/28-cloud-base`.

```
$ vagrant up
$ vagrant ssh
```

Once SSH'd in (RHCOS example):

```
$ sudo su
# cd /srv/fedora-coreos-login-messages
# rpm-ostree install rpms/noarch/*
# systemctl reboot
```

## Operation

Let `x` denote `{motd,issue}`.

- Symlinks from `/etc/x` to `/run/x` are set by `systemd-tmpfiles`.
- `issuegen` and `motdgen` generate `/run/x`, from files in `/etc/coreos/x.d`, `/run/coreos/x.d`, `/lib/usr/coreos/x.d`.
- Users may append to `issue` or `motd` by placing a file in `/etc/coreos/x.d/`.

## Directory tree (unpacking the rpm)

```
[root@a5cba1b23420 fedora-coreos-login-messages]# ./view-rpm-tree.sh

...

[root@a5cba1b23420 view-rpm-tree-output]# tree view-rpm-tree-output
[root@a5cba1b23420 fedora-coreos-login-messages]# tree view-rpm-tree-output/
view-rpm-tree-output/
|-- etc
|   `-- coreos
|       |-- issue.d
|       `-- motd.d
|-- run
|   `-- coreos
|       |-- issue.d
|       `-- motd.d
`-- usr
    |-- lib
    |   |-- coreos
    |   |   |-- issue.d
    |   |   |   `-- base.issue
    |   |   |-- issuegen
    |   |   |-- motd.d
    |   |   `-- motdgen
    |   |-- systemd
    |   |   `-- system
    |   |       |-- issuegen.path
    |   |       |-- issuegen.service
    |   |       |-- motdgen.path
    |   |       `-- motdgen.service
    |   |-- tmpfiles.d
    |   |   |-- coreos-profile.conf
    |   |   |-- issuegen.conf
    |   |   `-- motdgen.conf
    |   `-- udev
    |       `-- rules.d
    |           `-- 91-issuegen.rules
    `-- share
        |-- coreos
        |   `-- coreos-profile.sh
        |-- doc
        |   `-- coreos-ux
        |       `-- README.md
        `-- licenses
            `-- coreos-ux
                `-- LICENSE

24 directories, 14 files
```

## Next steps
- [x] account for issue in install.sh (for testing)
- [x] script to enable the systemd units and reboot (for testing)
- [x] script to configure PAM (for testing)
    - no script; make it a responsibility of the user, not of the package installer. therefore PAM doesn't need to be a dependency
- [x] make systemd-tmpfiles config to create symlinks
    - rpm installation can also set up symlinks, but it should be a responsibility of systemd to keep the symlinks maintained (user can easily override by placing config in /etc/\*.conf acting on top of /usr/lib/\*.conf)
- [x] wrap everything up with rpm spec file (includes gen scripts, tmpfiles config, systemd units)
- [x] show systemd failed units, or find out where this is currently being done https://github.com/coreos/init/commit/5e82c6bf46d746545281a219ce82af57e950f026#diff-892b6c24ac66bd41b13adeaeb077da83
- [ ] testing that the info we need shows in RHCOS
  - [ ] a "you should not be sshing into this OS" message in motd
  - [ ] a "dev info" message (motd and issue)
  - [ ] ssh keys in issue and motd (NOTE: ssh-keygen functionality will not be handled here)
  - [ ] added users in issue and motd
  - [x] ip address in issue
  - [ ] some info  on updates (booting, pending, etc) from rpm-ostree status --json? in motd
  - [x] failed units on login
- [ ] check installation against RHCOS and FCOS
- [ ] ensure licensing is correct

## Issues to figure out

- [ ] How to manage files existing at /etc/motd and /etc/issue before installing? If they exist, this causes problems when installing if they are included under `%files` as part of the coreos-ux package. The symlinks `/etc/motd -> /etc/run` and `/etc/issue -> /run/issue` do not get created if they exist.
- [ ] After a system update, how do motd/issue source the updated info? Possibly add a PathChanged to the appropriate system/\*.path unit file, so that motd and issue can update.
- [ ] How to make sure issuegen.* and motdgen.* are enabled (i.e. run every boot, and whenever something is dropped into a motd.d/issue.d) after installing? Done in `%post`? `WantedBy` a `.target` required? Or is this done by preset config?
- [ ] After installing the rpms generated by `rpm-build.sh` more tmpfiles named `pkg-coreos-ux-*.conf` are
created, which include lines to create directories in run; `/run/coreos`, `/run/coreos/issue.d`, `/run/coreos/motd.d`. This clutters up tmpfiles.d (given that this package contains 3 tmpfiles already). May want to consider another something like CL's [baselayout](https://github.com/coreos/baselayout/blob/master/tmpfiles.d/baselayout.conf) rather than have several tmpfiles.

## Enhancements for future
- have upstream PAM include the "trying" functionality, use this config rather than symlinks
- have upstream PAM search issue.d with pam_issue.so (rather than agetty, go through one interface - PAM)
