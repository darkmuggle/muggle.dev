# CoreOS Assembler

Running CoreOS Assembler in a clean environment is a must in my book. And as such, I've setup a couple of helpers.

The basic is around `cosa_cmd` which:
- setups the container
- makes `/tmp` a tmpfs
- adds the needed devices
- adds in some of my pet paths
- runs the shell and passes the command line

The real trick, though, is with `-a=stdin -a=stdout -a=stderr` which allows for interacting with the container.
```
cosa_cmd() {
   root="${1}"; shift;
   podman run \
      -a=stdin -a=stdout -a=stderr \
      --rm -i --tty --security-opt label=disable --privileged=true \
      --uidmap=1000:0:1 --uidmap=0:1:1000 --uidmap 1001:1001:64536 \
      --device /dev/fuse \
      --device /dev/kvm \
      --tmpfs /tmp \
      --name "${root}-$(openssl rand -hex 6)" \
      --volume=/var/tmp:/var/tmp \
      --volume=/home/ben/src/builder/${root}:/srv:z \
      --volume=/etc/koji.conf.d:/etc/koji.conf.d:ro \
      --volume=/etc/krb5.conf.d:/etc/krb5.conf.d:ro \
      --volume=/etc/pki:/etc/pki:ro \
      --volume=/home/ben/src:/src \
      --volume="${COREOS_COREOS_ASSEMBLER_GIT:-/home/ben/src/cosa}/src:/usr/lib/coreos-assembler:ro" \
      ${COREOS_ASSEMBLER_CONTAINER:-quay.io/coreos-assembler/coreos-assembler:latest} \
      shell -- "${@}"
}
```
The `${root}` is a path in `/home/ben/src/builder` for the variant I am building (such as Fedora CoreOS).

And then I have these helpers:
```
fcosa() { cosa_cmd fcos cosa ${@} ; }
fcos_run() { fcosa run --qemu-multipath --devshell-console ${@}; }
```
