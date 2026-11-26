# RHEL 10 Image Mode sealed-image lab

This lab builds on [Red Hat's official sealing example](https://github.com/redhat-cop/rhel-bootc-examples/tree/main/sealing) and demonstrates a RHEL 10.3 Image Mode host protected by:

- a signed Unified Kernel Image (UKI)
- UEFI Secure Boot with a lab-owned signature database (`db`) certificate
- a ComposeFS/fs-verity digest embedded in the UKI

In this example we build our Image Mode image from a RHEL 10 host and deploy it from a Fedora workstation with [bcvk](https://github.com/bootc-dev/bcvk/tree/main) and libvirt.
## What the lab proves

Secure Boot verifies the signed systemd-boot binary and UKI. The UKI contains the ComposeFS digest for the immutable operating-system tree; ComposeFS then uses fs-verity to validate the files it serves. In an extension activity we can prove a UKI signed by a different `db` key is rejected by the VM firmware before the OS starts.

## Lab topology

| System                  | Purpose                                                                                |
| ----------------------- | -------------------------------------------------------------------------------------- |
| Registered RHEL 10 host | Builds and pushes the sealed OCI image. Package mode and Image Mode are both suitable. |
| Quay.io                 | Stores the OCI image. Use your own organisation and repository.                        |
| Fedora workstation      | Runs `bcvk` and libvirt to create the VM.                                              |
| RHEL 10 sealed VM       | Boots the image with Secure Boot enabled and the lab certificate enrolled.             |

## Repository layout

Keep private material out of source control:

```text
.
├── Containerfile
├── README.md
└── target/
    └── keys/                 # private; add target/ to .gitignore
        ├── PK.key  PK.crt  PK.cer
        ├── KEK.key KEK.crt KEK.cer
        ├── db.key  db.crt  db.cer
        └── GUID.txt
```

Create `.gitignore` with at least:

```gitignore
target/
*.key
```

## 1. Create lab Secure Boot keys

Run on the **Build Host**. These are deliberately lab keys; do not
reuse them for production.

```bash
umask 077
install -d -m 0700 target/keys
cd target/keys
uuidgen --random > GUID.txt

openssl req -new -x509 -newkey rsa:2048 -nodes -sha256 -days 3650 \
  -subj '/CN=RHEL Image Mode Lab Platform Key/' \
  -keyout PK.key -out PK.crt
openssl x509 -in PK.crt -outform DER -out PK.cer

openssl req -new -x509 -newkey rsa:2048 -nodes -sha256 -days 3650 \
  -subj '/CN=RHEL Image Mode Lab Key Exchange Key/' \
  -keyout KEK.key -out KEK.crt
openssl x509 -in KEK.crt -outform DER -out KEK.cer

openssl req -new -x509 -newkey rsa:2048 -nodes -sha256 -days 3650 \
  -subj '/CN=RHEL Image Mode Lab Signature Database/' \
  -keyout db.key -out db.crt
openssl x509 -in db.crt -outform DER -out db.cer

# Copy the public certificate keys to the Fedora Host in a home user path
ssh <fedora-user>@<fedora-ip> install -d -m 0700 <path>/target/keys
scp PK.crt KEK.crt db.crt GUID.txt \
  <fedora-user>@<fedora-ip>:<path>/target/keys/

cd ../..

```

`db.key` signs the systemd-boot binary and UKI. `bcvk` uses the `PK`, `KEK`, and `db` certificates to create an OVMF variable store with the lab trust chain enrolled.

## 2. Prepare the Fedora virtualisation host

On the **Fedora Host** install `bcvk` and the local libvirt/KVM stack:

```bash
sudo dnf install -y bcvk libvirt-client libvirt-daemon-kvm qemu-kvm
sudo systemctl enable --now libvirtd
```

Confirm that the current user can use libvirt:

```bash
virsh -c qemu:///system list --all
```

If that requires elevated access, add the user to the appropriate local libvirt group, sign out, and sign in again.

Copy the *public* key directory to the Fedora workstation for `bcvk`. Do not copy it more widely than needed, and never publish the `.key` files.

## 3. Prepare the RHEL build host

Register the RHEL build host so it can pull `registry.redhat.io` content and use the supported RHEL repositories:

```bash
sudo rhc connect
sudo dnf install -y podman
podman login registry.redhat.io
podman login quay.io
```

Define reusable image references. Replace the placeholder namespace and image
name:

```bash
export REGISTRY=quay.io
# quay username
export NAMESPACE=example-org
# quay repo name
export IMAGE=rhel10-sealed-im
export TAG=0.1
export IMAGE_REF="$REGISTRY/$NAMESPACE/$IMAGE:$TAG"
```

## 4. Understand the Containerfile

Use the validated minimal sealed-image Containerfile supplied with this lab.
Its stages must remain in this order:

1. **`target-base`** creates the clean RHEL root filesystem
2. **`boot-rpms`** downloads `systemd-boot-unsigned` for offline installation
3. **`tools`** provides `systemd-ukify` only during the build
4. **`base-rootfs`** constructs the final OS tree and removes `bootupd` and `shim-x64`; the lab boots directly with systemd-boot and a signed UKI
5. **`penultimate`** adds immutable configuration, signs systemd-boot with `systemd-sbsign`, and clears mutable package bookkeeping under `/var`
6. **`sealer`** runs `bootc container ukify` against the `penultimate` tree. This computes the ComposeFS digest and embeds it in the signed UKI
7. The **final stage** copies only the UKI into `/boot/EFI/Linux/`

Do not add packages, a Quadlet, firewall configuration, or other files after the sealing stage. Any root-tree change requires a fresh UKI and signature. Make these changes earlier in the build.

The Containerfile must use RHEL 10.3 and pass the key material as build secrets (`podman build --secret id=`), never with `COPY` or `ARG`:

```Dockerfile
ARG base=registry.redhat.io/rhel10/rhel-bootc:10.3

# Later, when signing systemd-boot and creating the UKI:
RUN --mount=type=secret,id=secureboot_key \
    --mount=type=secret,id=secureboot_cert \
    # systemd-sbsign / bootc container ukify commands
```

The build command below supplies `target/keys/db.key` and `target/keys/db.crt`
to those secret mounts. The private key must not appear in an image layer.

## 5. Build and push the good image

From the repository root on the RHEL build host:

```bash
podman build --no-cache --pull=always \
  --cap-add=all \
  --security-opt=label=type:container_runtime_t \
  --device /dev/fuse \
  --secret id=secureboot_key,src=target/keys/db.key \
  --secret id=secureboot_cert,src=target/keys/db.crt \
  -t "$IMAGE_REF" .

podman push "$IMAGE_REF"
```
Don't forget the period at the end of the build command to specify the current working directory. Alternatively replace it with the path to the Containerfile using `-f <path-to-containerfile>`

**Optional:** Record the pushed immutable manifest digest for the lab notes:

```bash
skopeo inspect "docker://$IMAGE_REF" --format '{{.Digest}}'
```

Use a new tag for every build (`0.1`, `0.2`, and so on). Do not move an
existing tag to different content during validation.

## 6. Deploy as a Secure-Boot VM with bcvk

On the **Fedora workstation**, authenticate to Quay if the repository is private,
then deploy. Thanks to `bcvk` we do not need to convert the image to a qcow2:

```bash
podman login quay.io

# populate $IMAGE_REF with the details of the image; replacing NAMESPACE, IMAGE, TAG with your details
IMAGE_REF=quay.io/NAMESPACE/IMAGE:TAG

podman pull $IMAGE_REF

bcvk libvirt run \
  --ssh-wait \
  --name sealed-demo \
  --filesystem=ext4 \
  --secure-boot-keys target/keys \
  "$IMAGE_REF"
```
- The `--filesystem=ext4` flag instructs `bootc install` to use ext4 for the root filesystem, as it supports `fs-verity` unlike the default XFS
- The `--secure-boot-keys` target or keys flag directs `bcvk` to the directory with our PK, KEK, and db certificates, enabling `bcvk` to use `virt-fw-vars` to enroll these keys into the OVMF firmware variable store for Secure Boot
- The `--ssh-wait` flag causes `bcvk` to wait for SSH availability before it returns, confirming the system boot

Validate the booted deployment over SSH or from the console:

```bash
bcvk libvirt ssh sealed-demo
bootc status
bootctl status
cat /proc/cmdline | tr ' ' '\n' | grep '^composefs='
```

Expected outcome: `bootc status` shows the intended image and a `Verity`
value; the kernel command line contains `composefs=<sha512 digest>`; and
`bootctl status` reports Secure Boot enabled.

## Optional exercise: stage an image signed by the wrong key

This exercise proves that a successfully pulled and staged image is still blocked when its UKI is not signed by the certificate enrolled in UEFI. To restore the VM you will need access to a graphical console like [virt-manager](https://virt-manager.org/) to be able to interact with the UEFI boot interface.

### Create a deliberately untrusted signing key

Keep the original `PK`, `KEK`, and `db` files used by `bcvk`. On the **Build Host** create a second
key pair only for the image signature:

```bash
install -d -m 0700 target/bad-keys
openssl req -new -x509 -newkey rsa:2048 -nodes -sha256 -days 3650 \
  -subj '/CN=Deliberately Untrusted Lab Signing Key/' \
  -keyout target/bad-keys/db.key \
  -out target/bad-keys/db.crt
```

Build the same validated Containerfile with the alternate signing secrets and
a distinct tag, for example `0.2-bad`:

```bash
export BAD_REF="$REGISTRY/$NAMESPACE/$IMAGE:0.2-bad"

podman build --no-cache --pull=always \
  --cap-add=all \
  --security-opt=label=type:container_runtime_t \
  --device /dev/fuse \
  --secret id=secureboot_key,src=target/bad-keys/db.key \
  --secret id=secureboot_cert,src=target/bad-keys/db.crt \
  -t "$BAD_REF" .
  
podman push "$BAD_REF"
```

On the good VM, stage it and reboot:

```bash
sudo podman login quay.io --authfile /etc/ostree/auth.json
BAD_REF="$REGISTRY/$NAMESPACE/$IMAGE:0.2-bad"
sudo bootc switch --authfile /etc/ostree/auth.json "$BAD_REF"
sudo systemctl reboot
```

Open virt-manager
```
virt-manager --connect qemu:///session
```

Open the VM in virt-manager
![[img/virt-manager-sealed.png]]

Expected result: the UEFI firmware refuses to load the UKI, typically with an
`Access denied` message. This is the expected Secure Boot protection: the
firmware does not trust the alternate signing certificate.

![[img/bad-key-denial.png]]


### Recover the lab VM

1. Force reset the vm and when it is booting tap the escape key repeatedly until you see the below
![[img/boot-select.png]]
2. Select the alternate boot. This can take trial and error. It should be the `10.3-0` version
3. If the good key image is selected you will see a blank screen. Now you can go back to your Fedora host cli and ssh in using `bcvk libvirt ssh sealed-demo`
4. After booting the trusted deployment, verify it with:

   ```bash
   bootc status
   bootctl status
   ```

5. If `bootc status` reports a **Rollback image (queued)** for the bad image, run the following while booted into the known-good deployment to swap the order back and cancel that queued rollback state:

   ```bash
   sudo bootc rollback
   bootc status
   ```

This will remove the bad image from `queued` so that a reboot won't boot the bad image
![[img/bad-key-denial.png]]
6. Reboot and confirm that the trusted image remains the default.

Alternatively you could disable Secure Boot in the UEFI device manager which can be useful to diagnose a lab VM, but it permits the untrusted OS to boot. Do not use that as the normal recovery method and re-enable Secure Boot before treating the VM as trusted.

## Troubleshooting: ComposeFS digest mismatch

If `bootc switch` reports an error like the following, it has rejected the
image before it was queued or rebooted:

```text
The UKI ... embedded composefs= digest ... doesn't match any of 4 supported
xattr filtering mode/EROFS format version combinations
```

This is distinct from the wrong-key exercise. It means the ComposeFS digest in
the UKI does not match the root filesystem reconstructed from the OCI image.

In this lab, the working remediation was to revert unvalidated Containerfile
changes that added extra packages, a Quadlet, and offline firewalld state. Keep
the minimal, validated Containerfile as the baseline. Reintroduce changes one
at a time, seal every resulting image afresh, and test each image using a new
tag.

Useful diagnostics on the builder:

```bash
bootc --version
podman version
buildah version

image_id=$(podman image inspect "$IMAGE_REF" --format '{{.Id}}')
bootc container compute-composefs-digest-from-storage "$image_id"
```

If a ComposeFS digest mismatch occurs, do not reboot into the image. Preserve
the image digest, Containerfile, build logs, and the output above for support
investigation.

## References

- [RHEL 10: Cryptographic sealing of bootc images](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/cryptographic-sealing-of-bootc-images-technology-preview)
- [bcvk Secure Boot deployment](https://bootc.dev/blog/2026-may-07-sealed-images-deploying/)
- [bootc rollback behaviour](https://bootc.dev/bootc/man/bootc-rollback.8.html)
