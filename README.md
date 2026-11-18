# riscv_linux_ctr_demo
Simple demo to test CTR feature enabled on riscv

**Setup ENV**
export WORKSPACE=/<path-to-workspace>
export ROOTFS=$WORKSPACE/rootfs
export LINUX=$WORKSPACE/linux
export QEMU=$WORKSPACE/qemu
export SYSROOT=$ROOTFS/temp-rootfs
cd $WORKSPACE

**Build Linux Kernel Image**
git clone --depth=1 -b ctr_upstream -j8 https://github.com/rajnesh-kanwal/linux.git $LINUX
cd $LINUX
export ARCH=riscv
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
mkdir build
make O=./build defconfig
make O=./build -j$(nproc)
cd ..

**Build Root-fs**
mkdir $ROOTFS && cd $ROOTFS
wget https://raw.githubusercontent.com/rajnesh-kanwal/common_work/main/rootfs_related/create_rootfs.sh
chmod +x ./create_rootfs.sh
./create_rootfs.sh

**Build perf tool**
cd $LINUX/tools/perf
sudo -E PKG_CONFIG_LIBDIR="$SYSROOT/usr/lib/riscv64-linux-gnu/pkgconfig/" VF=1 make   EXTRA_CFLAGS="--sysroot=$SYSROOT"   ARCH=riscv  CROSS_COMPILE=riscv64-linux-gnu- NO_LIBBPF=1  prefix='$(SYSROOT)/usr' install
The above commands will create and install perf tool in our rootfs.

**Compile a basic test program**
cd $ROOTFS
cat >./test.c <<EOF
#include <sys/syscall.h>

#define N 1000000

void f2(void)
{
}

void f3(void)
{
}

void f1(unsigned long n)
{
        if (n & 1UL)
                f2();
        else
                f3();
}

int _start(void)
{
        unsigned long i = 0;
        for (i=0; i < N; i++)
                f1(i);
    
        asm volatile ("li a7, 93\n"
                      "ecall\n"
                      );
        __builtin_unreachable();
}
EOF

# Compile and copy the test file.
riscv64-unknown-linux-gnu-gcc -nostdlib test.c
sudo cp a.out $SYSROOT/root/

**Create rootfs.ext4**
cd $ROOTFS
dd if=/dev/zero of=rootfs.ext4 bs=1G count=4
mkfs.ext4 rootfs.ext4
mkdir ./tmp
sudo mount rootfs.ext4 ./tmp
sudo cp -rp ./temp-rootfs/* ./tmp/
sudo umount ./tmp

**Run the demo**
$QEMU/build/qemu-system-riscv64 -M virt,aia=aplic-imsic,aia-guests=5 -cpu rv64,smaia=true,ssaia=true,smcdeleg=true,ssccfg=true,smcntrpmf=true,sscofpmf=true,sscsrind=true,smcsrind=true,smctr=true,ssctr=true  -icount auto -m 8192 -nographic -kernel $LINUX/build/arch/riscv/boot/Image -append "root=/dev/vda  rw console=ttyS0 earlycon=sbi" -drive file=$ROOTFS/rootfs.ext4,format=raw,if=none,id=rootfs -device virtio-blk-pci,drive=rootfs  -netdev user,id=usernet,hostfwd=tcp:127.0.0.1:7722-0.0.0.0:22 -device e1000e,netdev=usernet
Username is root and password is riscv. Run perf.

perf record -e instructions:ppu -c 10000  -b ./a.out
perf report -b --stdio
Example output:

# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 779K of event 'instructions:ppu'
# Event count (approx.): 779285
#
# Overhead  Command  Source Shared Object  Source Symbol  Target Symbol  Basic Block Cycles
# ........  .......  ....................  .............  .............  ..................
#
    16.68%  a.out    a.out                 [.] _start     [.] f1         -                 
    16.68%  a.out    a.out                 [.] f1         [.] _start     -                 
    16.67%  a.out    a.out                 [.] f1         [.] f1         -                 
    16.64%  a.out    a.out                 [.] _start     [.] _start     -                 
     8.35%  a.out    a.out                 [.] f3         [.] f1         -                 
     8.34%  a.out    a.out                 [.] f1         [.] f3         -                 
     8.33%  a.out    a.out                 [.] f1         [.] f2         -                 
     8.30%  a.out    a.out                 [.] f2         [.] f1         -                 


#
# (Cannot load tips.txt file, please install perf!)
#
Run perf in RASEMU mode.

perf record --call-graph lbr -e instructions:ppu -c 10000  ./a.out
perf report -b --stdio
Example output:

# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 3K of event 'instructions:ppu'
# Event count (approx.): 30470000
#
# Children      Self  Command  Shared Object  Symbol    
# ........  ........  .......  .............  ..........
#
   100.00%    29.54%  a.out    a.out          [.] _start
            |          
             --70.46%--_start
                       f1
                       |          
                       |--11.55%--f2
                       |          
                        --11.42%--f3

    70.46%    47.49%  a.out    a.out          [.] f1
            |          
            |--47.49%--_start
            |          f1
            |          
             --22.97%--f1
                       |          
                       |--11.55%--f2
                       |          
                        --11.42%--f3

    11.55%    11.55%  a.out    a.out          [.] f2
            |
            ---_start
               f1
               f2

    11.42%    11.42%  a.out    a.out          [.] f3
            |
            ---_start
               f1
               f3


