FROM ubuntu:22.04

ARG CROSS_ARCH="arm64"
ARG CROSS_TRIPLET="aarch64-linux-gnu"

ENV LLVM_VER=22

WORKDIR /tmp

RUN <<\EOF
	set -eux

	# curl is needed for adding the repos.
	apt-get update
	apt-get install -y curl gnupg

	# Add LLVM repository for clang.
	curl -L https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -
	echo "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-${LLVM_VER} main" > /etc/apt/sources.list.d/llvm.list

	# Add CMake repo for newer versions of CMake
	curl -L https://apt.kitware.com/keys/kitware-archive-latest.asc | gpg --dearmor - > /usr/share/keyrings/kitware-archive-keyring.gpg
	echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' > /etc/apt/sources.list.d/kitware.list

	# Install host packages.
	# Install cross-compiler and qemu.
	apt-get update
	apt-get install -y --no-install-recommends autoconf automake build-essential clang-${LLVM_VER} lld-${LLVM_VER} llvm-${LLVM_VER} libc++-${LLVM_VER}-dev curl extra-cmake-modules git libasound2-dev libcurl4-openssl-dev libdbus-1-dev libdecor-0-dev libegl-dev libevdev-dev libfontconfig-dev libfuse2 libgtk-3-dev libgudev-1.0-dev libharfbuzz-dev libinput-dev libopengl-dev libpipewire-0.3-dev libpulse-dev libssl-dev libudev-dev libwayland-dev libx11-dev libx11-xcb-dev libxcb1-dev libxcb-composite0-dev libxcb-cursor-dev libxcb-damage0-dev libxcb-glx0-dev libxcb-icccm4-dev libxcb-image0-dev libxcb-keysyms1-dev libxcb-present-dev libxcb-randr0-dev libxcb-render0-dev libxcb-render-util0-dev libxcb-shape0-dev libxcb-shm0-dev libxcb-sync-dev libxcb-util-dev libxcb-xfixes0-dev libxcb-xinput-dev libxcb-xkb-dev libxext-dev libxkbcommon-x11-dev libxrandr-dev libxss-dev libtool ninja-build patchelf pkg-config unzip zlib1g-dev libegl1-mesa-dev libdrm-dev libgbm-dev appstream g++-${CROSS_TRIPLET} debootstrap binfmt-support qemu-user-static cmake
EOF

# Create chroot.
RUN debootstrap --arch="$CROSS_ARCH" jammy "/${CROSS_ARCH}-chroot"

RUN chroot "/${CROSS_ARCH}-chroot" <<\EOF
	set -eux
	sed -i 's/main/main universe/' /etc/apt/sources.list
	apt-get update -y
	apt-get install -y curl gnupg

	if [ "$CROSS_ARCH" != "armhf" ]; then
		# Add LLVM repository for libc++
		curl -L https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add -
		echo "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-${LLVM_VER} main" > /etc/apt/sources.list.d/llvm.list
		LIBCPP="libc++-${LLVM_VER}-dev"
	fi

	# Install packages
	apt-get update
	set +u
	apt-get install -y --no-install-recommends autoconf automake build-essential cmake extra-cmake-modules git libasound2-dev libcurl4-openssl-dev libdbus-1-dev libdecor-0-dev libegl-dev libevdev-dev libfontconfig-dev libgtk-3-dev libgudev-1.0-dev libharfbuzz-dev libinput-dev libopengl-dev libpipewire-0.3-dev libpulse-dev libssl-dev libudev-dev libwayland-dev libx11-dev libx11-xcb-dev libxcb1-dev libxcb-composite0-dev libxcb-cursor-dev libxcb-damage0-dev libxcb-glx0-dev libxcb-icccm4-dev libxcb-image0-dev libxcb-keysyms1-dev libxcb-present-dev libxcb-randr0-dev libxcb-render0-dev libxcb-render-util0-dev libxcb-shape0-dev libxcb-shm0-dev libxcb-sync-dev libxcb-util-dev libxcb-xfixes0-dev libxcb-xinput-dev libxcb-xkb-dev libxext-dev libxkbcommon-x11-dev libxrandr-dev libxss-dev libtool ninja-build pkg-config unzip zlib1g-dev libegl1-mesa-dev libdrm-dev libgbm-dev "$LIBCPP"
EOF

COPY armhf-toolchain.cmake .

# This block is for building LLVM for armhf
RUN <<\EOF
	set -eux
	if [ "$CROSS_ARCH" = "armhf" ]; then
		# Set toolchain file location
		TOOLCHAINFILE="$(realpath armhf-toolchain.cmake)"

		# Get LLVM SemVer rather than just the major version
		LLVM_SemVer=$(clang-${LLVM_VER} --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -n1)

		# Download LLVM source to build libc++ for armhf
		curl -LO https://github.com/llvm/llvm-project/releases/download/llvmorg-${LLVM_SemVer}/llvm-project-${LLVM_SemVer}.src.tar.xz
		tar xf llvm-project-${LLVM_SemVer}.src.tar.xz
		cd llvm-project-${LLVM_SemVer}.src
		
		# Enable execution of armhf binaries because of hardcoded python execution in LLVM
		export QEMU_LD_PREFIX="/${CROSS_ARCH}-chroot/"

		# Install prebuilt native development files for LLVM tools
		apt-get install -y --no-install-recommends libclang-${LLVM_VER}-dev
		
		# Build libc++ and install to chroot
		cmake -S llvm -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_HOST_TRIPLE="$CROSS_TRIPLET" -DLLVM_DEFAULT_TARGET_TRIPLE="$CROSS_TRIPLET" -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind" -DLIBUNWIND_ENABLE_SHARED=OFF -DLLVM_VER="$LLVM_VER" -DCMAKE_TOOLCHAIN_FILE="$TOOLCHAINFILE" -DCMAKE_INSTALL_PREFIX="/${CROSS_ARCH}-chroot/usr"
		ninja -C build install
		mv -t /armhf-chroot/lib/arm-linux-gnueabihf/ /armhf-chroot/lib/arm-unknown-linux-gnueabihf/*
	fi
EOF

# Clean cache and WORKDIR
RUN rm -rf /var/cache/apt/archives /var/lib/apt/lists/* /tmp/*
# Clean chroot cache
RUN rm -rf /${CROSS_ARCH}-chroot/var/cache/apt/archives /${CROSS_ARCH}-chroot/var/lib/apt/lists/*
