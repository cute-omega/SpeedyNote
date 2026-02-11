# =============================================================================
# SpeedyNote Android Build Environment
# =============================================================================
# This Dockerfile creates a reproducible environment for building SpeedyNote
# for Android arm64-v8a and armeabi-v7a (32-bit).
#
# Contents:
#   - Ubuntu 22.04 LTS base
#   - Android SDK (API 35) + NDK r27
#   - Qt 6.9.3 for Android arm64-v8a + armv7 (32-bit)
#   - Qt 6.9.3 host tools (Linux x86_64)
#   - CMake, Ninja, OpenJDK 17
#
# Build:
#   docker build -t speedynote-android:latest .
#
# Run:
#   docker run -it --rm -v $(pwd)/..:/workspace speedynote-android:latest
#
# =============================================================================

FROM ubuntu:22.04

LABEL maintainer="SpeedyNote Developers"
LABEL description="Android build environment for SpeedyNote"

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# =============================================================================
# System Dependencies
# =============================================================================
RUN apt-get update && apt-get install -y --no-install-recommends \
    # Build tools
    build-essential \
    cmake \
    ninja-build \
    pkg-config \
    # Version control
    git \
    # Download tools
    wget \
    curl \
    unzip \
    # Java (required for Android SDK)
    openjdk-17-jdk \
    # Python (for aqtinstall)
    python3 \
    python3-pip \
    python3-venv \
    # SSL certificates (for downloads)
    ca-certificates \
    # Useful utilities
    file \
    && rm -rf /var/lib/apt/lists/*

# =============================================================================
# Environment Variables
# =============================================================================
ENV ANDROID_SDK_ROOT=/opt/android-sdk
ENV ANDROID_HOME=${ANDROID_SDK_ROOT}
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

# Qt version to install
# Qt 6.9.3 is the recommended version:
#   - 6.7.x: Has keyboard crash bug (QtEditText.m_optionsChanged NullPointerException)
#   - 6.9.3: Works correctly ✓
#   - 6.10.x: Has OpenGL threading deadlock on Android
ENV QT_VERSION=6.9.3
ENV QT_ROOT=/opt/qt
ENV QT_ANDROID=${QT_ROOT}/${QT_VERSION}/android_arm64_v8a
ENV QT_ANDROID_ARMV7=${QT_ROOT}/${QT_VERSION}/android_armv7
# Note: aqtinstall uses "linux_gcc_64" as arch name but installs to "gcc_64"
ENV QT_HOST=${QT_ROOT}/${QT_VERSION}/gcc_64

# NDK version
# Qt 6.10.x requires NDK r27+ for API 35 support (androidx.core:core:1.16.0 dependency)
ENV NDK_VERSION=27.2.12479018
ENV ANDROID_NDK_ROOT=${ANDROID_SDK_ROOT}/ndk/${NDK_VERSION}

# API levels
# Qt 6.10+ requires compileSdk 35+ due to androidx dependencies
ENV ANDROID_MIN_SDK=26
ENV ANDROID_TARGET_SDK=35

# Add tools to PATH
ENV PATH="${QT_HOST}/bin:${ANDROID_SDK_ROOT}/cmdline-tools/latest/bin:${ANDROID_SDK_ROOT}/platform-tools:${PATH}"

# =============================================================================
# Install Android SDK & NDK
# =============================================================================
RUN mkdir -p ${ANDROID_SDK_ROOT}/cmdline-tools && \
    # Download Android command-line tools
    wget -q -O /tmp/cmdline-tools.zip \
        "https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip" && \
    unzip -q /tmp/cmdline-tools.zip -d ${ANDROID_SDK_ROOT}/cmdline-tools && \
    mv ${ANDROID_SDK_ROOT}/cmdline-tools/cmdline-tools ${ANDROID_SDK_ROOT}/cmdline-tools/latest && \
    rm /tmp/cmdline-tools.zip

# Accept licenses
RUN yes | sdkmanager --licenses > /dev/null 2>&1 || true

# Install SDK components
RUN sdkmanager --install \
    "platforms;android-${ANDROID_TARGET_SDK}" \
    "build-tools;35.0.0" \
    "ndk;${NDK_VERSION}" \
    "platform-tools"

# =============================================================================
# Install Qt via aqtinstall
# =============================================================================
# Create virtual environment for aqtinstall to avoid pip externally-managed error
RUN python3 -m venv /opt/aqt-venv && \
    /opt/aqt-venv/bin/pip install --no-cache-dir aqtinstall

# Install Qt for Android (arm64-v8a)
RUN /opt/aqt-venv/bin/aqt install-qt linux android ${QT_VERSION} android_arm64_v8a \
    -O ${QT_ROOT} \
    --modules qtimageformats

# Install Qt for Android (armeabi-v7a / 32-bit ARM)
RUN /opt/aqt-venv/bin/aqt install-qt linux android ${QT_VERSION} android_armv7 \
    -O ${QT_ROOT} \
    --modules qtimageformats

# Install Qt host tools (needed for moc, uic, rcc, etc.)
# Note: For Qt 6.x, architecture is "linux_gcc_64" not "gcc_64"
RUN /opt/aqt-venv/bin/aqt install-qt linux desktop ${QT_VERSION} linux_gcc_64 \
    -O ${QT_ROOT} \
    --modules qtimageformats

# =============================================================================
# Verify Toolchain Files
# =============================================================================
# Explicitly check for the existence of Qt Android toolchain files.
# These are critical for CMake builds and must be present after installation.
RUN echo "=== Checking Qt Android Toolchain Files ===" && \
    TOOLCHAIN_ARM64="${QT_ANDROID}/lib/cmake/Qt6/qt.toolchain.cmake" && \
    TOOLCHAIN_ARMV7="${QT_ANDROID_ARMV7}/lib/cmake/Qt6/qt.toolchain.cmake" && \
    echo "Checking: ${TOOLCHAIN_ARM64}" && \
    if [ -f "${TOOLCHAIN_ARM64}" ]; then \
        echo "  ✓ arm64-v8a toolchain file exists"; \
    else \
        echo "  ✗ ERROR: arm64-v8a toolchain file NOT FOUND"; \
        echo "  Expected location: ${TOOLCHAIN_ARM64}"; \
        echo "  This will cause CMake builds to fail!"; \
        exit 1; \
    fi && \
    echo "Checking: ${TOOLCHAIN_ARMV7}" && \
    if [ -f "${TOOLCHAIN_ARMV7}" ]; then \
        echo "  ✓ armeabi-v7a toolchain file exists"; \
    else \
        echo "  ✗ ERROR: armeabi-v7a toolchain file NOT FOUND"; \
        echo "  Expected location: ${TOOLCHAIN_ARMV7}"; \
        echo "  This will cause CMake builds to fail!"; \
        exit 1; \
    fi && \
    echo "All toolchain files verified successfully."

# =============================================================================
# Verify Installation and Print Diagnostics
# =============================================================================
RUN echo "" && \
    echo "=============================================================================" && \
    echo "Build Environment Diagnostic Information" && \
    echo "=============================================================================" && \
    echo "" && \
    echo "--- Qt Installation Paths ---" && \
    echo "Qt Version: ${QT_VERSION}" && \
    echo "Qt Root: ${QT_ROOT}" && \
    echo "Qt Android arm64-v8a: ${QT_ANDROID}" && \
    echo "Qt Android armeabi-v7a: ${QT_ANDROID_ARMV7}" && \
    echo "Qt Host (Linux x86_64): ${QT_HOST}" && \
    echo "" && \
    echo "--- Qt Android Toolchain Files ---" && \
    echo "arm64-v8a toolchain: ${QT_ANDROID}/lib/cmake/Qt6/qt.toolchain.cmake" && \
    ls -lh "${QT_ANDROID}/lib/cmake/Qt6/qt.toolchain.cmake" && \
    echo "armeabi-v7a toolchain: ${QT_ANDROID_ARMV7}/lib/cmake/Qt6/qt.toolchain.cmake" && \
    ls -lh "${QT_ANDROID_ARMV7}/lib/cmake/Qt6/qt.toolchain.cmake" && \
    echo "" && \
    echo "--- Android SDK and NDK ---" && \
    echo "ANDROID_SDK_ROOT: ${ANDROID_SDK_ROOT}" && \
    echo "ANDROID_NDK_ROOT: ${ANDROID_NDK_ROOT}" && \
    echo "NDK Version: ${NDK_VERSION}" && \
    echo "Installed SDK Platforms:" && \
    ls -1 ${ANDROID_SDK_ROOT}/platforms/ && \
    echo "Installed NDK Versions:" && \
    ls -1 ${ANDROID_SDK_ROOT}/ndk/ && \
    echo "NDK Location:" && \
    ls -ld ${ANDROID_NDK_ROOT} && \
    echo "" && \
    echo "--- Build Tools ---" && \
    echo "CMake:" && \
    cmake --version | head -1 && \
    which cmake && \
    echo "" && \
    echo "Ninja:" && \
    ninja --version && \
    which ninja && \
    echo "" && \
    echo "Java:" && \
    java --version | head -1 && \
    echo "JAVA_HOME: ${JAVA_HOME}" && \
    echo "" && \
    echo "--- Qt Host Tools ---" && \
    echo "qmake:" && \
    ls -lh ${QT_HOST}/bin/qmake && \
    ${QT_HOST}/bin/qmake --version && \
    echo "" && \
    echo "androiddeployqt:" && \
    ls -lh ${QT_HOST}/bin/androiddeployqt && \
    (${QT_HOST}/bin/androiddeployqt --version 2>&1 | head -3) || echo "  (version output not available)" && \
    echo "" && \
    echo "moc:" && \
    ls -lh ${QT_HOST}/bin/moc && \
    (${QT_HOST}/bin/moc -v 2>&1) || echo "  (version output not available)" && \
    echo "" && \
    echo "--- Sample Qt Android Binaries ---" && \
    echo "Qt Android arm64-v8a binaries (sample):" && \
    ls ${QT_ANDROID}/bin/ | head -5 && \
    echo "Qt Android armeabi-v7a binaries (sample):" && \
    ls ${QT_ANDROID_ARMV7}/bin/ | head -5 && \
    echo "" && \
    echo "=============================================================================" && \
    echo "Build Environment Ready" && \
    echo "=============================================================================" && \
    echo ""

# =============================================================================
# Workspace Setup
# =============================================================================
WORKDIR /workspace

# Default command
CMD ["/bin/bash"]

