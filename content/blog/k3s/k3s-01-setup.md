---
title: "K3s part 1: Setup your workstation"
date: 2020-12-11T00:01:00-06:00
tags: ['k3s']
---

## Prepare your workstation

When working with kubernetes, you should resist the urge to directly login into
the host server via SSH, unless you have to. Instead, you will create all the
config files on your local laptop, which will be referred to as your
workstation, and use `kubectl` to access the remote cluster API.

You will need to install several command line tools on your workstation:

 * A modern BASH shell, being the default Linux terminal shell, but also
   available on various platforms.
 * `kubectl`:
   * Arch Linux: `sudo pacman -S kubectl`
   * Other OS: [See docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-using-native-package-management)
   * You can take a small detour now and setup bash shell tab-key completion for
     kubectl, this is quite useful for interactive use. Run `kubectl completion
     -h` and follow the directions for setting up your shell. However, you can
     skip this, since all of the commands you will run are already prepared for
     you in this blog.
       
 * `kustomize`:
   * Arch Linux: `sudo pacman -S kustomize`
   * Other OS: [see releases](https://github.com/kubernetes-sigs/kustomize/releases)
   * You may have heard about kubectl having kustomize as a built-in feature
     (`kubectl apply -k`). However, the version of kustomize that is bundled
     with kubectl is old, and has bugs. You should use the latest version of
     `kustomize` directly instead of the bundled kubectl version (`kustomize
     build | kubectl apply -f - `).
   * NOTICE: [kustomize v3.9.0 does not
     work](https://github.com/kubernetes-sigs/kustomize/issues/3340#issue-761638279)
     (Fails to create `RoleBinding` resources) This may already be fixed in a
     later version, but the tested WORKING version as of 2020-12-14 is kustomize
     [v3.8.8](https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize%2Fv3.8.8)
     
 * `kubeseal`:
 
   * Arch Linux has an [AUR build](https://aur.archlinux.org/packages/kubeseal/)
   * Other OS: [see releases](https://github.com/bitnami-labs/sealed-secrets/releases)
   * Note: only install the client side at this point, you will install the
     cluster side later in a different way.

 * `flux`:
 
   * Arch Linux has an [AUR build](https://aur.archlinux.org/packages/flux-go/)
   * Other OS: [see docs](https://github.com/fluxcd/flux2/tree/main/install)
   * Optional: Add shell completion support to your `~/.bashrc`
   
```env-static
## Optional bash shell completion for flux
. <(flux completion bash)
```

 * `git`:
   * Arch Linux: `sudo pacman -S git`
   * Ubuntu: `sudo apt install git`
   * [Other OS](https://git-scm.com/downloads)
     
## Running Commands

This blog is written in a Literate Programming style, containing *exact* BASH
shell commands for you to paste into your workstation terminal.

These block-quoted commands are intended to be copy-and-pasted directly into
your terminal *without editing them*. Commands that need configuration, will
reference environment variables, which you create before you run the command, so
that you may customize the variables first, and then run the command as-is.

You should configure your BASH so that your pasted commands are never run unless
you give your confirmation, after pasting, by pressing the Enter key. This also
allows you the opportunity to edit the commands on the terminal prompt line,
(which you will only need to do in the case of customizing variables, not for
editing commands). Run this to enable this feature in BASH:

```bash
# Enable for the current shell:
bind 'set enable-bracketed-paste on'
# Enable for future environments:
echo "set enable-bracketed-paste on" >> ${HOME}/.inputrc
```

For example, here is a command block that is asking you to customize two
environment variables (`SOME_VARIABLE` and `SOME_OTHER_VARIABLE`). You should
copy and paste this into your shell, and edit the values on the command line
(`foo` and `bar`; change them to something else), then press Enter.

```env
SOME_VARIABLE=foo
SOME_OTHER_VARIABLE=bar
```

Now run a command that references the variables. You don't need to edit it, just
copy the command, paste into the terminal, and press Enter:

```bash
echo some command that needs ${SOME_VARIABLE} and ${SOME_OTHER_VARIABLE}
```

Often, a command will use the [BASH HEREDOC
format](https://tldp.org/LDP/abs/html/here-docs.html) to create whole files
without needing to use a text editor. For example, this next code block will
create a new temporary file with a random name, with the contents `Hello,
World!`. 

```bash
TMP_FILE=$(mktemp)
cat <<EOF > ${TMP_FILE}
Hello, World!
EOF

echo "-------------------------------------------"
echo The random temporary file is ${TMP_FILE}
echo The contents written were: $(cat ${TMP_FILE})
```

The contents of the file is the line(s) between `cat <<EOF` on line 2 and the
second `EOF` on its own line, line 4 (`Hello, World!\n`). Any lines that comes
after the second `EOF` (the `echo` lines) are just regular commands, not part of
the content of the file created. (Technically, HEREDOC format allows any marker
instead of `EOF` but this blog will always use `EOF` by convention, which is
mnemonic for `End Of File`.)

Note that the previous example rendered environment variables *before* writing
the file. The file contains the *value* of the variable as it was at the time of
creation, and replaces the variable name reference. In order to write a shell
script, via HEREDOC, that contains variable *names* (not values), you need to
disable this behaviour. To do this, you put quotes around the first `EOF`
marker, and then no variables will be substituted in the body:

```bash
TMP_FILE=$(mktemp --suffix .sh)
cat <<'EOF' > ${TMP_FILE}
## I'm a shell script that needs raw variable names to be preserved.
## To do this, the HEREDOC used <<'EOF' instead of <<EOF
echo Hello I am ${USER} on ${HOSTNAME} at $(date)
EOF

echo "-------------------------------------------"
cat ${TMP_FILE}
echo "The variables are evaluated on run:"
sh ${TMP_FILE}
```

## Create a local git repository

You need a place to store your cluster configuration, so create a git repository
someplace on your workstation called `flux-infra` (or whatever you want to call
it). The `flux-infra` repository will manage the root level of one or more of
your clusters. Each cluster storing its manifests in its own sub-directory,
listed by domain name. Each kubernetes namespace gets a sub-sub-directory :
 * `~/git/flux-infra/${CLUSTER}/${NAMESPACE}` 
 
Choose the directory where to create the git repo and the domain name for your
new cluster:

```env
FLUX_INFRA_DIR=${HOME}/git/flux-infra
CLUSTER=k3s.example.com
```

```bash
mkdir -p ${FLUX_INFRA_DIR}/${CLUSTER} && \
git -C ${FLUX_INFRA_DIR} init && \
cd ${FLUX_INFRA_DIR}/${CLUSTER} && \
echo Cluster working directory: $(pwd)
```

In an upcoming post, you will create a git remote to push this repository to.