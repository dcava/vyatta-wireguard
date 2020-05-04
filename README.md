#Please see the new "official" home for vyatta-ubnt modules at https://github.com/WireGuard/wireguard-vyatta-ubnt

# vyatta-wireguard

This is a Vyatta module and pre-built binaries for the Ubiquiti EdgeRouter
to support [WireGuard](https://www.wireguard.io/).

### Installation

Download the [latest release](https://github.com/Lochnair/vyatta-wireguard/releases) for your model (or build it yourself
here with `make`) and then install it via:

    $ sudo dpkg -i ./wireguard-octeon-${RELEASE}.deb
    or
    $ sudo dpkg -i ./wireguard-ralink-${RELEASE}.deb

After you'll be able to have a `wireguard` section in `interfaces`.

### Usage

You can learn about how to actually use WireGuard on the
[website](https://www.wireguard.io/). All of the concepts translate
evenly here. Here's an example vyatta configuration:


```
configure

set interfaces wireguard wg0 address 192.168.33.1/24
set interfaces wireguard wg0 listen-port 51820
set interfaces wireguard wg0 route-allowed-ips true

set interfaces wireguard wg0 peer GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc= endpoint example1.org:29922
set interfaces wireguard wg0 peer GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc= allowed-ips 192.168.33.101/32

set interfaces wireguard wg0 peer aBaxDzgsyDk58eax6lt3CLedDt6SlVHnDxLG2K5UdV4= endpoint example2.net:51820
set interfaces wireguard wg0 peer aBaxDzgsyDk58eax6lt3CLedDt6SlVHnDxLG2K5UdV4= allowed-ips 192.168.33.102/32
set interfaces wireguard wg0 peer aBaxDzgsyDk58eax6lt3CLedDt6SlVHnDxLG2K5UdV4= allowed-ips 192.168.33.103/32

set interfaces wireguard wg0 private-key /config/auth/wg.key

commit
```

If you prefer not to put private keys in the config file, the `private-key` and `preshared-key` items can alternatively take a file path on the filesystem, such as one in `/config/auth/`.

### Routing

Currenty there is no integration between the routing daemon and WireGuard which means allowed-ips for a peer will not be updated based upon dynamic routing updates. If you are going to utilize a dynamic routing protocol over wireguard interfaces it is recommended to configure them with a single peer per interface, disable route-allowed-ips and either configure allowed-ips to 0.0.0.0/0 or all ip addresses which might ever be routed over the interface including any multicast addresses required by the routing protocol.
.
### Binaries

This repository ships prebuilt binaries, made from the [WireGuard source code](https://git.zx2c4.com/WireGuard/tree/src/). If you're buliding from scratch, please be sure to use `-mabi=64` in your `CFLAGS` for compiling the userspace tools; otherwise there will be strange runtime errors. The binaries in this repository are statically linked against [musl libc](https://www.musl-libc.org/) to mitigate potential issues with Ubiquiti's outdated libc.

#### Kernel Module

```bash
$ mkdir -p linux-src/linux-3.10
$ cd linux-src/linux-3.10
$ tar xf ../../source/kernel_*.tgz
$ cd ../..
$ tar xf source/cavm-executive_*.tgz
$ export KERNELDIR="$PWD/linux-src/linux-3.10/kernel"
$ cd OCTEON-SDK
$ . ./env-setup OCTEON_CN50XX --no-runtime-model --verbose
$ cd ..
$ export CROSS_COMPILE=mips64-octeon-linux-gnu-
$ export CC=mips64-octeon-linux-gnu-gcc
$ export ARCH=mips
$ make -C linux-src/linux-3.10/kernel -j$(nproc)
$ git clone https://git.zx2c4.com/WireGuard
$ make -C linux-src/linux-3.10/kernel M=$PWD/WireGuard/src modules -j$(nproc)
$ ls -l WireGuard/src/wireguard.ko
```

#### Userspace Tools

```bash
$ cd OCTEON-SDK
$ . ./env-setup OCTEON_CN50XX --no-runtime-model --verbose
$ cd ..
$ export CROSS_COMPILE=mips64-octeon-linux-gnu-
$ export CC=mips64-octeon-linux-gnu-gcc
$ export ARCH=mips
$ mkdir -p prefix
$ prefix="$PWD/prefix"
$ git clone git://git.musl-libc.org/musl
$ cd musl
$ CFLAGS=-mabi=64 ./configure --prefix="$prefix" --disable-shared
$ make -j$(nproc)
$ make install
$ cd ..
$ make -C linux-src/linux-3.10/kernel headers_install INSTALL_HDR_PATH="$prefix"
$ git clone git://git.netfilter.org/libmnl
$ cd libmnl
$ ./autogen.sh
$ CC="$prefix/bin/musl-gcc" CFLAGS=-mabi=64 ./configure --prefix="$prefix" --disable-shared --enable-static --host=x86_64-pc-linux-gnu
$ make -j$(nproc)
$ make install
$ cd ..
$ git clone https://git.zx2c4.com/WireGuard
$ CC="$prefix/bin/musl-gcc" make -C WireGuard/src/tools/ -j$(nproc)
$ ls -l WireGuard/src/tools/wg
```
