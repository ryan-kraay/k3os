ARG REPO
ARG TAG
FROM ubuntu:disco

RUN apt-get update -y && \
    apt-get install -y \
        kmod initramfs-tools curl rsync xz-utils \
        build-essential autoconf automake libtool \
        gawk alien fakeroot dkms libblkid-dev \
        uuid-dev libudev-dev libssl-dev zlib1g-dev \
        libaio-dev libattr1-dev libelf-dev \
        python3 python3-dev python3-setuptools \
        python3-cffi libffi-dev flex

ARG ARCH
ENV KVERSION=5.0.0-43-generic
ENV URL=https://github.com/rancher/k3os-kernel/releases/download/5.0.0-43.47-rancher1
ENV KERNEL_XZ=${URL}/kernel-generic_${ARCH}.tar.xz
ENV KERNEL_EXTRA_XZ=${URL}/kernel-extra-generic_${ARCH}.tar.xz
ENV KERNEL_HEADERS_XZ=${URL}/kernel-headers-generic_${ARCH}.tar.xz

ENV ZFS_VERSION=0.8.3
ENV ZFS_URL=https://github.com/openzfs/zfs/releases/download/zfs-${ZFS_VERSION}/zfs-${ZFS_VERSION}.tar.gz

# Download kernel
RUN mkdir -p /usr/src
RUN curl -fL $KERNEL_XZ -o /usr/src/kernel.tar.xz
RUN curl -fL $KERNEL_EXTRA_XZ -o /usr/src/kernel-extra.tar.xz
RUN curl -fL $KERNEL_HEADERS_XZ -o /usr/src/kernel-headers.tar.xz

# Extract to /usr/src/root
RUN mkdir -p /usr/src/root && \
    cd /usr/src/root && \
    tar xvf /usr/src/kernel.tar.xz && \
    tar xvf /usr/src/kernel-extra.tar.xz && \
    tar xvf /usr/src/kernel-headers.tar.xz && \
    cd usr/src/linux-headers-${KVERSION} && \
    ln -s ./x86 ./arch/amd64

# Download zfs
RUN cd /usr/src && curl -OL ${ZFS_URL}

# Build zfs
RUN mkdir -p /usr/src/zfs && \
    cd /usr/src && \
    tar zxvf zfs-${ZFS_VERSION}.tar.gz --strip-components=1 -C zfs && \
    cd zfs && \
    sed -i 's|depmod -ae -F|depmod -b $(DESTDIR)$(INSTALL_MOD_PATH) -C $(DESTDIR)/etc/depmod.conf -ae -F|' ./module/Makefile.in && \
    ./autogen.sh && \
    ./configure \
        --with-linux=/usr/src/root/usr/src/linux-headers-${KVERSION} && \
    make -s -j${nproc} V=1

# Install zfs
RUN cd /usr/src/zfs && \
    make DESTDIR=/usr/src/root install

# Create initrd
RUN mkdir /usr/src/initrd && \
    rsync -a /usr/src/root/lib/ /lib/ && \
    depmod $KVERSION && \
    mkinitramfs -k $KVERSION -o /usr/src/initrd.tmp

# Generate initrd firmware and module lists
RUN mkdir -p /output/lib && \
    mkdir -p /output/headers && \
    cd /usr/src/initrd && \
    gzip -dc /usr/src/initrd.tmp | cpio -idmv && \
    find lib/modules -name \*.ko > /output/initrd-modules && \
    echo lib/modules/${KVERSION}/modules.order >> /output/initrd-modules && \
    echo lib/modules/${KVERSION}/modules.builtin >> /output/initrd-modules && \
    find lib/firmware -type f > /output/initrd-firmware && \
    find usr/lib/firmware -type f | sed 's!usr/!!' >> /output/initrd-firmware

# Copy output assets
RUN cd /usr/src/root && \
    cp -r usr/src/linux-headers* /output/headers && \
    cp -r lib/firmware /output/lib/firmware && \
    cp -r lib/modules /output/lib/modules && \
    cp boot/System.map* /output/System.map && \
    cp boot/config* /output/config && \
    cp boot/vmlinuz-* /output/vmlinuz && \
    echo ${KVERSION} > /output/version
