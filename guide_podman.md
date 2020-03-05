# tina "workbenches"
### 2020.03.04

It is no secret that I'm a Harry Potter fan. My work computer is named `porpentina` after the dark wizard catcher (aka aurua) Porpentina "Tina" Goldstien from the Fantastic Beast franchise.

My "tina" workbenches are special containers setup for various tasks.

## tina ephmeral workbenches

For work environments, I have my own series of "pet containers.

The biggest problem that I encountered with the pet containers is that I do my work on several computers and then use Mosh and TMUX to connect; running `podman run -it...` results in a fairly mono-chrome experience.

### dockerfile

I'd publish my Dockerfile, but suffice it to say, it contains $CORP bits.

In the course of my day, I interact with services that uses Kerberos and/or use TLS connections to services that use an internal CA. Like any proper Muggle, the entire workbench is described in a Dockerfile for what I call the "dark-toolbox."

In the Dockerfile I do have a section that creates an internal user that matches my external user.

```
FROM fedora:latest

# Work uses Kerberos Auth, so copy that in too.
COPY krb5.conf /etc/
COPY krb5.conf.d/* /etc/krb5.conf.d/

# Update my certificates...
COPY certs/* /etc/pki/ca-trust/source/anchors/
RUN find /etc/pki/ca-trust/source/anchors -type f && \
    update-ca-trust extract && \
    update-ca-trust enable && \

RUN dnf -y update --refresh  && \
    dnf install -y \
        dumbinit \
        sudo \
        <other stuff>

# Make sure that I can run sudo in my container.
RUN mkdir -p /etc/sudoers.d && \
    chmod u+s /usr/bin/sudo && \
    echo "%wheel        ALL=(ALL)       NOPASSWD: ALL" > /etc/sudoers.d/container

# Disable error about sudo not being able to change the RLimit
RUN echo "Set disable_coredump false" > /etc/sudo.conf
RUN chmod 0644 /etc/sudo.conf

# Add my user
RUN useradd  \
    --uid 1000 \
    --user-group \
    --groups users,adm,wheel,kvm,kvm78,kvm124,kvm232 ben

WORKDIR /home/ben
USER ben

# Allow the container to run detatched
ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["sleep", "infinity"]
```

#### Why not use the Fedora-Toolbox?

Initially I _did_ use the Fedora Toolbox. However, I found that I need more customization and it was a bit buggy. Besides, I was more interested in learning how it was doing what it does.

#### Building it

Since this container is customized to my needs, I build it _my_ user's permissions.

```
buildah bud \
    --userns-uid-map=1000:0:1 --userns-uid-map=0:1:1000 --userns-uid-map 1001:1001:64536 \
    --userns-gid-map=1000:0:1 --userns-gid-map=0:1:1000 --userns-gid-map 1001:1001:64536 \
    -f Dockerfile \
    --security-opt label=disable \
    --net=host \
    -t localhost/dark-toolbox \
    --squash
```

The key part is the `--user-uid-map` and `--user-gid-map`. See the `Running it` section below for the reason why.

### Running it

The following is a snipet from my `~/.bashrc`. I simply run `tina` and get myself a pretty terminal..

```
tina() {
    hname="tina-$(openssl rand -hex 3)"
    podman run \
        --rm=true \
        -it \
        --uidmap=1000:0:1 --uidmap=0:1:1000 --uidmap 1001:1001:64536 \
        --security-opt label=disable \
        --privileged \
        --device /dev/fuse \
        --device /dev/kvm \
        --net=host \
        --device=/dev/kvm \
        --cap-drop=AUDIT_WRITE,MKNOD,NET_RAW \
        --name "${hname}" --hostname "${hname}" \
        --env=TERM=xterm-256color \
        --env=COLUMNS=$(tput cols) \
        --env=LINES=$(tput lines) \
        --volume=/etc/pki:/etc/pki:ro \
        --volume=/home/ben:/home/ben \
        --entrypoint='["/bin/bash", "--rcfile", "~/.bashrc", "--login"]' \
        localhost/dark-toolbox
}
```

#### user-uid-map

Like the `buildah` step above, the key part is `--user-uid-map` and `--user-gid-map`. This ensure that files created by either `root` or the user `ben` will be owned by me (uid 1000) outside the container.

This shows me creating a file `test` in the container
```
ben@tina.dev-ff97e9:~/tmp $ id
uid=1000(ben) gid=1001(ben) groups=1001(ben) context=unconfined_u:system_r:container_runtime_t:s0

ben@tina.dev-ff97e9:~/tmp $ touch test

ben@tina.dev-ff97e9:~/tmp $ ls -lh test
-rw-rw-r--. 1 ben ben 0 Mar  4 17:18 test
```

And this shows that `test` is owned by me _outside_ the container.
```
ben@porpentina.local:~/tmp $ ls -lh
total 0
-rw-rw-r--. 1 ben 101000 0 Mar  4 10:18 test
```

#### What about the group?

Honestly, I have figured that out yet.
