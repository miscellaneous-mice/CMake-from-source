## Bootstrap Compiler (Temporary Spark)
```
mkdir -p ~/tools/bootstrap_compiler && cd ~/tools/bootstrap_compiler
wget --no-check-certificate https://toolchains.bootlin.com/downloads/releases/toolchains/x86-64/tarballs/x86-64--glibc--stable-2025.08-1.tar.xz
tar -xf x86-64--glibc--stable-2025.08-1.tar.xz

export PATH="$HOME/tools/bootstrap_compiler/x86-64--glibc--stable-2025.08-1/bin:$PATH"
export CC=x86_64-linux-gcc
export CXX=x86_64-linux-g++
export MY_LOCAL="$HOME/local"
mkdir -p $MY_LOCAL
```

## GNU Make Installation
```
cd ~/tools
wget --no-check-certificate https://ftp.gnu.org/gnu/make/make-4.4.1.tar.gz
tar -xzf make-4.4.1.tar.gz && cd make-4.4.1

./configure --prefix=$MY_LOCAL --disable-dependency-tracking
./build.sh
./make install
# Refresh path to use your NEW make
export PATH="$MY_LOCAL/bin:$PATH"
```

## OpenSSL (For CMake HTTPS Support)
```
cd ~/tools
wget --no-check-certificate https://github.com/openssl/openssl/releases/download/openssl-3.3.0/openssl-3.3.0.tar.gz
tar -xzf openssl-3.3.0.tar.gz && cd openssl-3.3.0
./config --prefix=$MY_LOCAL --openssldir=$MY_LOCAL/ssl
make -j$(nproc)
make install
```

## CMake Installation
```
cd ~/tools
wget https://github.com/Kitware/CMake/releases/download/v3.31.0/cmake-3.31.0.tar.gz
tar -xzf cmake-3.31.0.tar.gz && cd cmake-3.31.0

# Point to the OpenSSL you just built
./bootstrap --prefix=$MY_LOCAL -- -DOPENSSL_ROOT_DIR=$MY_LOCAL -DCMAKE_USE_OPENSSL=ON
make -j$(nproc)
make install
```

## GCC Installation
```
cd ~/tools
export BOOTSTRAP_SYSROOT="$HOME/tools/bootstrap_compiler/x86-64--glibc--stable-2025.08-1/x86_64-buildroot-linux-gnu/sysroot"

wget --no-check-certificate https://ftp.gnu.org/gnu/gcc/gcc-14.2.0/gcc-14.2.0.tar.gz
tar -xzf gcc-14.2.0.tar.gz && cd gcc-14.2.0

# Pre-download math libs to bypass SSL certificate errors in the script
wget --no-check-certificate -P . https://gcc.gnu.org/pub/gcc/infrastructure/gmp-6.2.1.tar.bz2
wget --no-check-certificate -P . https://gcc.gnu.org/pub/gcc/infrastructure/mpfr-4.1.0.tar.bz2
wget --no-check-certificate -P . https://gcc.gnu.org/pub/gcc/infrastructure/mpc-1.2.1.tar.gz
wget --no-check-certificate -P . https://gcc.gnu.org/pub/gcc/infrastructure/isl-0.24.tar.bz2
./contrib/download_prerequisites

cd ..
mkdir -p gcc-build && cd gcc-build

../gcc-14.2.0/configure \
    --prefix=$MY_LOCAL \
    --with-sysroot=$BOOTSTRAP_SYSROOT \
    --with-build-sysroot=$BOOTSTRAP_SYSROOT \
    --enable-languages=c,c++ \
    --disable-multilib

make -j$(nproc)
make install

# Copy sysroot to permanent home
mkdir -p ~/local/sysroot
cp -a $BOOTSTRAP_SYSROOT/. ~/local/sysroot/
```

## Add this to the VERY BOTTOM of `~/.bashrc`
```
# --- 1. Base Paths ---
export MY_LOCAL="$HOME/local"
export PATH="$MY_LOCAL/bin:$PATH"

# --- 2. Compiler Selection ---
export CC="$MY_LOCAL/bin/gcc"
export CXX="$MY_LOCAL/bin/g++"

# --- 3. Internal GCC Search Paths (The "Secret Sauce") ---
# These ensure GCC finds stdio.h and standard C++ headers in your sysroot automatically
export C_INCLUDE_PATH="$MY_LOCAL/sysroot/usr/include"
export CPLUS_INCLUDE_PATH="$MY_LOCAL/sysroot/usr/include:$MY_LOCAL/include/c++/14.2.0"

# --- 4. Build Tool Hints (CPPFLAGS & LDFLAGS) ---
# These help CMake/Make find your CUSTOM built libraries like OpenSSL
export CPPFLAGS="-I$MY_LOCAL/include"
export LDFLAGS="-L$MY_LOCAL/lib64 -L$MY_LOCAL/lib -Wl,-rpath,$MY_LOCAL/lib64 -Wl,-rpath,$MY_LOCAL/lib"

# --- 5. Library Runtime Paths ---
export LD_LIBRARY_PATH="$MY_LOCAL/lib64:$MY_LOCAL/lib:$LD_LIBRARY_PATH"
export LIBRARY_PATH="$MY_LOCAL/lib64:$MY_LOCAL/lib:$LIBRARY_PATH"

# --- 6. Metadata Path ---
export PKG_CONFIG_PATH="$MY_LOCAL/lib64/pkgconfig:$MY_LOCAL/lib/pkgconfig:$PKG_CONFIG_PATH"
```
