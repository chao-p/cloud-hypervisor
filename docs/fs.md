# How to use virtio-fs

In the context of virtualization, it is always convenient to be able to share a directory from the host with the guest.

__virtio-fs__, also known as __vhost-user-fs__ is a virtual device defined by the VIRTIO specification which allows any VMM to perform filesystem sharing.

## Pre-requisites

### The daemon

This virtual device relies on the _vhost-user_ protocol, which assumes the backend (device emulation) is handled by a dedicated process running on the host. This daemon is called __virtiofsd__ and needs to be present on the host.

_Install virtiofsd_
```bash
VIRTIOFSD_URL="$(curl --silent https://api.github.com/repos/intel/nemu/releases/latest | grep "browser_download_url" | grep "virtiofsd-x86_64" | grep -o 'https://.*[^ "]')"
wget --quiet $VIRTIOFSD_URL -O "virtiofsd"
chmod +x "virtiofsd"
sudo setcap cap_sys_admin+epi "virtiofsd"
```
_Create shared directory_
```bash
mkdir /tmp/shared_dir
```
_Run virtiofsd_
```bash
./virtiofsd \
    -d \
    -o vhost_user_socket=/tmp/virtiofs \
    -o source=/tmp/shared_dir \
    -o cache=none
```
The `cache=none` option here is an important one as it tells the daemon not to try any memory mapping of the files, but instead to use the _virtqueues_ to convey the files content. The support for the memory mapping of the files will be added later.

### The kernel

In order to leverage __virtio-fs__ support from within the guest, and because the code has not been merged in upstream Linux kernel yet, it is required to build a custom kernel embedding the patches.

The following branch `virtio-pmem_and_virtio-fs` on the repository https://github.com/sboeuf/linux.git includes all the needed patches to support __virtio-fs__.

Make sure to build a kernel out of this branch that can be then used to boot the VM.

## How to share directories with cloud-hypervisor

### Start the VM
Once the daemon is running, the option `--fs` from __cloud-hypervisor__ needs to be used.

Direct kernel boot option is preferred since we need to provide the custom kernel including the __virtio-fs__ patches. We could boot from `hypervisor-fw` if we had previously edited the image to replace the kernel binary.

Because _vhost-user_ expects a dedicated process (__virtiofsd__ in this case) to be able to access the guest RAM to communicate through the _virtqueues_ with the driver running in the guest, `--memory` option needs to be slightly modified. It needs to specify a backing file for the memory so that an external process can access it.

Assuming you have `clear-kvm.img` and `custom-vmlinux.bin` on your system, here is the __cloud-hypervisor__ command you need to run:
```bash
./cloud-hypervisor \
    --cpus 4 \
    --memory "size=512,file=/dev/shm" \
    --disk clear-kvm.img \
    --kernel custom-vmlinux.bin \
    --cmdline "console=ttyS0 reboot=k panic=1 nomodules root=/dev/vda3" \ 
    --fs tag=virtiofs,sock=/tmp/virtiofs,num_queues=1,queue_size=512
```

### Mount the shared directory
The last step is to mount the shared directory inside the guest, using the `virtio_fs` filesystem type.
```bash
mkdir mount_dir
mount \
    -t virtio_fs /dev/null mount_dir/ \
    -o tag=virtiofs,rootmode=040000,user_id=0,group_id=0
```
The `tag` needs to be consistent with what has been provided through the __cloud-hypervisor__ command line.
