<br />
<div align="center">
    <img src="lfs-logo.png" alt="Logo" width="200" height="80">

  <h3 align="center">A Simplified Tutorial for Building and OS Using Linux from Scratch</h3>

  <p align="center">
    If you have ever felt the itch to compile code for countless hours to build 
    <br> an operating system from the ground up, then using LFS is the right choice for you!<br> <br>
    I originally started this as a school project to make a custom OS, but ended up later going a different direction.
    <br> <b>Note: This tutorial is not complete, but will take you most of the way through the process.</b>
    <br /> <br> This documentation is going to end up being fairly long so that all that you need to do is copy/paste terminal commands.  Some of the commands are <b>VERY</b> large, and I’m sure it will be helpful to have them all in one place.  If some of the commands don’t work because of updates to the packages, you will have to diagnose the problem and refer to updated LFS documentation here.
    <a href="https://www.linuxfromscratch.org/lfs/view/stable/index.html">
    <strong><br>LFS Documentation</strong></a>
    <br />
  </p>
</div>



## STEP 1: Preparing the Host System

### Prerequisites
First, you’ll want to spin up an Ubuntu virtual machine. Generally speaking, you will probably run into fewer problems if you use an older distribution like 16.04, or 18.04.  You’ll also need a separate storage device with approximately 16GB, whether that be a USB or a virtual hard-drive.


Start your Ubuntu system, with your storage device connected and open the terminal.  We need to make sure the system is ready to work well with Linux From Scratch. 

### Getting to Work
There is a large block command found at this link (included in case it changes from what I have documented here):
<a href="https://www.linuxfromscratch.org/lfs/view/stable/chapter02/hostreqs.html"><strong><br>LFS First Command</strong></a>
<br> <br>
Copy that big block of text to create the file version-check.sh<br>
<br> Here is that command:

  ```sh
  $ cat > version-check.sh << "EOF"
#!/bin/bash
# A script to list version numbers of critical development tools

# If you have tools installed in other directories, adjust PATH here AND
# in ~lfs/.bashrc (section 4.4) as well.

LC_ALL=C 
PATH=/usr/bin:/bin

bail() { echo "FATAL: $1"; exit 1; }
grep --version > /dev/null 2> /dev/null || bail "grep does not work"
sed '' /dev/null || bail "sed does not work"
sort   /dev/null || bail "sort does not work"

ver_check()
{
   if ! type -p $2 &>/dev/null
   then 
     echo "ERROR: Cannot find $2 ($1)"; return 1; 
   fi
   v=$($2 --version 2>&1 | grep -E -o '[0-9]+\.[0-9\.]+[a-z]*' | head -n1)
   if printf '%s\n' $3 $v | sort --version-sort --check &>/dev/null
   then 
     printf "OK:    %-9s %-6s >= $3\n" "$1" "$v"; return 0;
   else 
     printf "ERROR: %-9s is TOO OLD ($3 or later required)\n" "$1"; 
     return 1; 
   fi
}

ver_kernel()
{
   kver=$(uname -r | grep -E -o '^[0-9\.]+')
   if printf '%s\n' $1 $kver | sort --version-sort --check &>/dev/null
   then 
     printf "OK:    Linux Kernel $kver >= $1\n"; return 0;
   else 
     printf "ERROR: Linux Kernel ($kver) is TOO OLD ($1 or later required)\n" "$kver"; 
     return 1; 
   fi
}

# Coreutils first because --version-sort needs Coreutils >= 7.0
ver_check Coreutils      sort     8.1 || bail "Coreutils too old, stop"
ver_check Bash           bash     3.2
ver_check Binutils       ld       2.13.1
ver_check Bison          bison    2.7
ver_check Diffutils      diff     2.8.1
ver_check Findutils      find     4.2.31
ver_check Gawk           gawk     4.0.1
ver_check GCC            gcc      5.2
ver_check "GCC (C++)"    g++      5.2
ver_check Grep           grep     2.5.1a
ver_check Gzip           gzip     1.3.12
ver_check M4             m4       1.4.10
ver_check Make           make     4.0
ver_check Patch          patch    2.5.4
ver_check Perl           perl     5.8.8
ver_check Python         python3  3.4
ver_check Sed            sed      4.1.5
ver_check Tar            tar      1.22
ver_check Texinfo        texi2any 5.0
ver_check Xz             xz       5.0.0
ver_kernel 4.19

if mount | grep -q 'devpts on /dev/pts' && [ -e /dev/ptmx ]
then echo "OK:    Linux Kernel supports UNIX 98 PTY";
else echo "ERROR: Linux Kernel does NOT support UNIX 98 PTY"; fi

alias_check() {
   if $1 --version 2>&1 | grep -qi $2
   then printf "OK:    %-4s is $2\n" "$1";
   else printf "ERROR: %-4s is NOT $2\n" "$1"; fi
}
echo "Aliases:"
alias_check awk GNU
alias_check yacc Bison
alias_check sh Bash

echo "Compiler check:"
if printf "int main(){}" | g++ -x c++ -
then echo "OK:    g++ works";
else echo "ERROR: g++ does NOT work"; fi
rm -f a.out

if [ "$(nproc)" = "" ]; then
   echo "ERROR: nproc is not available or it produces empty output"
else
   echo "OK: nproc reports $(nproc) logical cores are available"
fi
EOF

bash version-check.sh
  ```
<br>



Next, to check the needed packages:
```sh
$ bash version-check.sh
```
<br>


Here are a few more packages you will probably need:
```sh
$ sudo apt install bison gawk gcc g++ m4 make texinfo
```
<br>

Create a symbolic link pointing to bash
```sh
$ sudo ln -sf bash /bin/sh
```
<br>

Maybe run
```sh
$ bash version-check.sh
```
To make sure all host system requirements have been met.


## Step 2: Partitions

### Prerequisites

You will need a separate storage device. Add another virtual hard disk or use a USB drive
We will need approximately 16 GB of storage.  We will be making 3 partitions. A Boot Partition, a Swap Partition, and a GRUB BIOS Partition.


### Getting to Work

Use
```sh
$ lsblk
```
To find the path for the storage device you will be using. Usually for Ubuntu, it will be /dev/sdb or /dev/vdb

Time to Partition!
```sh
$ sudo cfdisk /path/
```
or 
```sh
$ sudo cfdisk /dev/sdb
```
You will enter a menu:
<ol>
Select “gpt” from the menu <br>
Select “new” <br>
Enter “100M” into partition size (this is for the boot partition) <br>
Move cursor to “free space” <br>
Select “new” again <br>
Enter “8G” (this is for the main partition) <br>
Create another new partition with the remaining space for the Swap partition <br> 
Select “write” from the menu and type yes <br>
Select “quit” <br><br>
</ol>

You can use 
```sh
$ lsblk
```
To double check the partitions were created
<br>

Time to format the partitions. Make sure to use the proper filepath
```sh
$ sudo mkfs -v -t ext2 /dev/sdb1
$ sudo mkfs -v -t ext4 /dev/sdb2
$ sudo mkswap /dev/sdb3
```


## Step 3: Mounting Disks

### Getting to Work

Make a working directory for LFS
```sh
$ sudo mkdir /mnt/lfs
$ export LFS=/mnt/lfs
```

The following command should return /mnt/lfs
(just to make sure those previous commands worked)
```sh
$ echo $LFS
```

<strong>Note: The LFS variable will reset upon restarting the system so these steps will have to be repeated if the host system is turned off and on again... In other words, you gotta make the whole rest of the OS from here or else start over again from this point if you lose power or progress for any reason.</strong>

Mount the main partition
```sh
$ sudo mount -v -t ext4 /dev/sdb2 $LFS     
```
<strong>SECOND WARNING: THIS WILL NEED TO BE REMOUNTED UPON RESTART OF THE SYSTEM</strong>

Make a directory for the boot partition
```sh
$ sudo mkdir $LFS/boot
```
Mount that directory to the boot partition or something
```sh
$ sudo mount -v -t ext2 /dev/sdb1 $LFS/boot
```

move to /mnt if necessary
```sh
$ cd /mnt   
```
The enter this command:
```sh
$ sudo /sbin/swapon -v /dev/sdb3
```

## Step 4: Downloading Packages and Patches

### Getting to Work

These commands are a little more straightforward. Enter the following:

```sh
$ cd /mnt/lfs
```
```sh
$ sudo mkdir sources
```
```sh
$ cd sources
```
```sh
$ sudo wget https://linuxfromscratch.org/lfs/view/stable/wget-list --no-check-certificate
```

LFS Documentation recommends that you make folders “sticky”, which we can do with the following:
```sh
$ sudo chmod -v a+wt $LFS/sources
```
```sh
$ sudo wget --input-file=wget-list --no-check-certificate
```

## Step 5: Make tools directory and add an LFS user

### Getting to Work

(To get to /mnt/lfs)
```sh
$ cd ..
```
Then:
```sh
$ sudo mkdir -v tools
```
Then link it to the system…
```sh
$ sudo ln -sv $LFS/tools /
```
<strong> WARNING: TOOLS DIRECTORY DISAPPEARS UPON RESTART </strong>

Time to make the LFS user
```sh
$ sudo groupadd lfs
```
```sh
$ sudo useradd -s /bin/bash -g lfs -m -k dev/null lfs
```

Set a password for the LFS user by entering the following command and entering a password:

```sh
$ sudo passwd lfs
```
(Choose password and then Chown with the following:)
```sh
$ sudo chown -v lfs $LFS/sources
```
Login as lfs user
```sh
$ su - lfs
```

Time to set up our environment (see https://www.linuxfromscratch.org/lfs/view/stable/chapter04/settingenvironment.html for more details)

Enter this command:
```sh
$ cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF
```
and then this one:
```sh
$ cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF
```

Then finally:
```sh
$ source ~/.bash_profile
```

Now we are ready to start compiling!

## Step 6: Compiling the OS... This is going to take longer than you think

### Getting to Work (Starting with Binutils)

Move directories
```sh
$ cd $LFS/sources
```

Then enter:
```sh
$ tar -xf binutils-2.39.tar.xz
```
(or whatever version of binutils you have in the current directory)

Go into new binutils directory

```sh
$ cd binutils-2.39
```
```sh
$ cd binutils-2.39
```
```sh
$ cd build
```
<br>

Now we are going to paste in configure commands from <br>
https://www.linuxfromscratch.org/lfs/view/stable/chapter05/binutils-pass1.html

Here is that command:
```sh
$ ../configure --prefix=$LFS/tools \
             --with-sysroot=$LFS \
             --target=$LFS_TGT   \
             --disable-nls       \
             --enable-gprofng=no \
             --disable-werror
```

Then we do the compile command "make" or 
```sh
$ make -j4
```
to make the compilation go a little faster. The normal make command for this compile is only about 2 minutes for my SBU.

Oops! I forgot we need to chown a directory, so let's do that.

```sh
$ exit
```
then
```sh
$ ln /mnt/lfs
```
and
```sh
$sudo chown -v lfs $LFS/ (tools)
```

Okay lets log back in as the lfs user
```sh
$ su - lfs
```
```sh
$ cd /mnt/lfs/sources/binutils-2.39/build
```


Now we need to make a symbolic link.  The documentation here did not work for me (https://www.linuxfromscratch.org/lfs/view/stable/chapter05/gcc-pass1.html), <br>
but I found another command from a youtube video that I have now misplaced.

Here is that command:
```sh
$ case $(uname -m) in
    Mkdir -v /tools/lib && ln -sv lib /tools/lib64
  ;;
esac
```

Now we
```sh
$ make install
```

then go back a directory
```sh
$ cd ..
```

and then we can delete binutils
```sh
$ cd ..rm -rf
```

Great! We have compiled our first package... Did you think we were done?

## Step 7: Compiling GCC

### Getting to Work 

Here's our first few commands:
```sh 
$ In lfs user
```
```sh
$ /mnt/lfs/sources directory
```
Unzip the GCC package
```sh
$ tar -xvf gcc-12.2.0.tar.xz
```
(or whatever version of gcc you have)
<br><br>

Go into the new directory
```sh
$ cd gcc-12.2.0
```

Move around some of the tar balls:
```sh
tar -xf ../mpfr-4.1.0.tar.xz
mv -v mpfr-4.1.0 mpfr
tar -xf ../gmp-6.2.1.tar.xz
mv -v gmp-6.2.1 gmp
tar -xf ../mpc-1.2.1.tar.gz
mv -v mpc-1.2.1 mpc
```

Move to build
```sh
mkdir -v build
cd       build
```

Then enter this configure statement:
```sh
../configure                  \
    --target=$LFS_TGT         \
    --prefix=$LFS/tools       \
    --with-glibc-version=2.36 \
    --with-sysroot=$LFS       \
    --with-newlib             \
    --without-headers         \
    --disable-nls             \
    --disable-shared          \
    --disable-multilib        \
    --disable-decimal-float   \
    --disable-threads         \
    --disable-libatomic       \
    --disable-libgomp         \
    --disable-libquadmath     \
    --disable-libssp          \
    --disable-libvtv          \
    --disable-libstdcxx       \
    --enable-languages=c,c++
```
Time to build! (and wait probably for several hours for it to compile... and if it fails we get to start all over again from the warning earlier in the tutorial. :D)
```sh
$ make
```
```sh
$ make install
```
```sh
$ cd ..
```
```sh
$ cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h
```

Now we can remove GCC after it has finished compiling. Gcc-12.2.0 is in /mnt/lfs/sources
```sh
$ cd ..
```
```sh
$ rm -rf gcc-12.2.0
```
## Step 7: Linux API Headers.. JK lets do GLIBC

Technically, It think the proper thing is to do Linux API headers next... but there are a lot of packages so maybe I will come back to it. Here's what I started with though.


### LINUX API HEADERS
```sh
In /mnt/lfs/sources
```
```sh
$ tar -xvf linux-5.19.2.tar.xz
```
```sh
$ cd linux-5.19.2
```
```sh
$make mrproper
```
```sh
$ make headers
```
```sh
$ find usr/include -type f ! -name '*.h' -delete
```
```sh
$ cp -rv usr/include $LFS/usr (permission denied and I can’t sudo…so i guess just delete this command?) 
```
This worked on UBUNTU 16.04
```sh
$ make INSTALL_GDR_PATH=dest headers_install...Something but I cannot get this to work right now

```
TO BE CONTINUED SOMEDAY


### GLIBC

You might kinda get the picture here. We unpack the tarball and then compile the code with certain configurations.  Here is a list of commands for the next package. (GLBIC)

```sh
$ cd ..
$ tar -xvf glibc-2.36.tar.xz
$ rm -rf linux-5.19.2
$ cd glibc-2.36 
$ patch -Np1 -i ../glibc-2.36-fhs-1.patch
$ mkdir -v build
$ cd build
$ echo "rootsbindir=/usr/sbin" > configparms
```

Then the following configuration command:
```sh
../configure                             \
      --prefix=/usr                      \
      --host=$LFS_TGT                    \
      --build=$(../scripts/config.guess) \
      --enable-kernel=3.2                \
      --with-headers=$LFS/usr/include    \
      libc_cv_slibdir=/usr/lib
```


and then our beautiful make command:
```sh
$ make
```
and another
```sh
$ make DESTDIR=$LFS install
```

This command also as per LFS documentation:
```sh
sed '/RTLDLIST=/s@/usr@@g' -i $LFS/usr/bin/ldd

echo 'int main(){}' | gcc -xc -
readelf -l a.out | grep ld-linux
```

After running previous commands there should be the following result:
```sh
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

IF this result appears, you can remove the output file:
```sh
$ rm -v a.out
```

To finalize the glibc section run this last command:
```sh
$LFS/tools/libexec/gcc/$LFS_TGT/12.2.0/install-tools/mkheaders
```

Awesome! That one was not too bad.

## Step 8: Libstdc++

This package is in the gcc tarball, so run the following command:

```sh
$ tar -xvf gcc-12.2.0.tar.xz
```

Enter the new directory we unpackaged:
```sh
$ cd gcc-12.2.0
```

Create build directory and enter:
```sh
$ mkdir -v build
$ cd build
```

Prepare libstdc++ for compilation:

```sh
../libstdc++-v3/configure           \
    --host=$LFS_TGT                 \
    --build=$(../config.guess)      \
    --prefix=/usr                   \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/12.2.0
```

Then compile:
```sh
$ make
```

and install
```sh
$ Make DESTDIR=$LFS install
```

Remove the libtool archive files because they are harmful for cross compilation:
```sh
$ rm -v $LFS/usr/lib/lib{stdc++,stdc++fs,supc++}.la
```

<br><br>
### <Strong>THE NEXT STEP IS TO INSTALL BINUTILS AND GCC AGAIN.</strong>
Which means exactly what you think it means. Go back and do steps 6 and 7 again before going on to step 8. All of these packages have dependencies that are needed to install others and then they overwrite themselves and the dependencies need to be reinstalled or somesuch. It's a fun time!

## Step 8: INSTALL M4 (no build directory)

This one's actually pretty straightforward.

First the tarball:
```sh
$ tar - xvf m4-1.4.19.tar.xz
$ cd m4-1.4.19
```
Then the configure command:
```sh
$ ./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
```
Then make
```sh
$ make
```
And install
```sh
$ make DESTDIR=$LFS install
```

Cool. They should all be like this.

## Step 9: INSTALL Ncurses (no cd into a build directory this time either)

Get that tarball again and just make the build directory:
```sh
$ tar -xvf ncurses-6.3.tar.gz
$ cd ncurses-6.3
$ sed -i s/mawk// configure
$ mkdir build
$ pushd build
```
Here's the configure command:
```sh
./configure
  make -C include
  make -C progs tic
popd
```
and another:
```sh
$ ./configure --prefix=/usr                \
            --host=$LFS_TGT              \
            --build=$(./config.guess)    \
            --mandir=/usr/share/man      \
            --with-manpage-format=normal \
            --with-shared                \
            --without-normal             \
            --with-cxx-shared            \
            --without-debug              \
            --without-ada                \
            --disable-stripping          \
            --enable-widec
```
then compile:
```sh
$ make
```
and install:
```sh
$ make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install
$ echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so
```

## Step 10: INSTALL Ncurses INSTALL BASH ( no build directory…)

Tarball:
```sh
$ tar -xvf bash-5.1.16.tar.z
$ cd bash-5.1.16
```
Configure:
```sh
$ ./configure --prefix=/usr                   \
            --build=$(support/config.guess) \
            --host=$LFS_TGT                 \
            --without-bash-malloc
```
Compile:
```sh
$ make
```
Install:
```sh
$ make DESTDIR=$LFS install
```

Then maybe do this too:
```sh
$ cd /mnt/lfs
$ mkdir bin
$ cd bin
$ ln -sv bash $LFS/bin/sh
```


## Step 11: To Be Continued...

I'm pretty sure you have to do another pass of GCC again, and seeing as that took my setup something like 1.5 hours every time to compile GCC, and there are still a lot of packages to go, I figured I had learned my lesson that this is a pretty serious undertaking.

Between the failed compilations, system crashes, and not having unlimited uptime for my system causing massive delays I went with another easier OS creation method. 

But, I will keep this documantation in case I need it in the future.  Who knows, maybe it will help someone else!