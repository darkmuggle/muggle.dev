# tina "workbenches"

It is no secret that I'm a Harry Potter fan. My work computer is named `porpentina` after the dark wizard catcher (aka aurua) Porpentina "Tina" Goldstien from the the Fanstastic Beast franchize.

My "tina" workbenches are special containers setup for various tasks.

## tina ephmeral workbenches

For work environments, I have my own serieis of "pet containers.

The biggest problem that I encountered with the pet containers is that I do my work on several computers and then use Mosh and TMUX to connect; running `podman run -it...` results in a fairly mono-chrome experience.

### dockerfile

TBD

### `.bashrc`

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
