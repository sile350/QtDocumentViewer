# Document Viewer Installation Guide

This is a comprehensive guide for installing and building Document Viewer on various platforms.

## 📋 Table of Contents

- [Requirements](#requirements)
- [Installing Qt](#installing-qt)
- [Building on Windows](#building-on-windows)
- [Building on Linux](#building-on-linux)
- [Building on macOS](#building-on-macos)
- [CMake Options](#cmake-options)
- [Troubleshooting](#troubleshooting)

## ⚙️ Requirements

### Required Components

- **Qt 6.8** or newer
  - Qt6::Core
  - Qt6::Gui
  - Qt6::Widgets

- **CMake 3.16** or newer

- **C++17 Compiler**:
  - Windows: MSVC 2019+ or MinGW
  - Linux: GCC 9+ or Clang 10+
  - macOS: Xcode 11+ (Clang)

### Optional Components

For full functionality, install:

- **Qt6::PrintSupport** - for document printing
- **Qt6::Pdf** and **Qt6::PdfWidgets** - for PDF viewing
- **Qt6::Quick3D** - for 3D viewer

## 📦 Installing Qt

### Option 1: Qt Online Installer (Recommended)

1. Download [Qt Online Installer](https://www.qt.io/download-qt-installer)
2. Run the installer
3. Create a Qt account (free for Open Source)
4. Select components:
   - Qt 6.8.x
   - Compiler for your platform
   - Qt Creator (optional)
   - Qt Debug Information Files (optional)
   - Qt Sources (optional)
5. In Qt components select:
   - Desktop (core modules)
   - Qt PDF
   - Qt Quick 3D

### Option 2: Package Managers

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install qt6-base-dev qt6-pdf-dev qt6-quick3d-dev cmake build-essential
```

#### Fedora
```bash
sudo dnf install qt6-qtbase-devel qt6-qtpdf-devel qt6-qtquick3d-devel cmake gcc-c++
```

#### Arch Linux
```bash
sudo pacman -S qt6-base qt6-pdf qt6-quick3d cmake gcc
```

#### macOS (Homebrew)
```bash
brew install qt@6 cmake
```

#### Windows (vcpkg)
```bash
vcpkg install qt6-base qt6-pdf qt6-quick3d
```

## 🪟 Building on Windows

### Using Qt Creator

1. Open Qt Creator
2. File → Open File or Project
3. Select `CMakeLists.txt` from project root
4. Select kit (e.g., Desktop Qt 6.8.0 MSVC2022 64bit)
5. Click **Configure Project**
6. Build → Build All (Ctrl+Shift+B)
7. Run the application (Ctrl+R)

### Using Command Line (MSVC)

```cmd
:: Open Developer Command Prompt for VS 2022

:: Navigate to project directory
cd path\to\documentviewer

:: Create build directory
mkdir build
cd build

:: Configure project
cmake .. -G "Visual Studio 17 2022" -A x64 -DCMAKE_PREFIX_PATH=C:\Qt\6.8.0\msvc2022_64

:: Build project
cmake --build . --config Release

:: Run application
app\Release\documentviewer.exe
```

### Using Command Line (MinGW)

```cmd
:: Ensure MinGW is in PATH

cd path\to\documentviewer
mkdir build
cd build

cmake .. -G "MinGW Makefiles" -DCMAKE_PREFIX_PATH=C:\Qt\6.8.0\mingw_64

cmake --build . --config Release

app\documentviewer.exe
```

### Creating Portable Application (Windows)

```cmd
:: After building
cd build\app\Release

:: Copy necessary DLLs
windeployqt documentviewer.exe --release --no-translations

:: Now you can copy the entire folder
```

## 🐧 Building on Linux

### Ubuntu/Debian

```bash
# Install dependencies
sudo apt update
sudo apt install qt6-base-dev qt6-pdf-dev qt6-quick3d-dev cmake build-essential git

# Clone repository
git clone https://github.com/username/documentviewer.git
cd documentviewer

# Create build directory
mkdir build
cd build

# Configure project
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build project (use all cores)
cmake --build . -j$(nproc)

# Run application
./app/documentviewer

# Optional: install to system
sudo cmake --install . --prefix /usr/local
```

### Fedora

```bash
# Install dependencies
sudo dnf install qt6-qtbase-devel qt6-qtpdf-devel qt6-qtquick3d-devel cmake gcc-c++ git

# Further steps same as Ubuntu
```

### Arch Linux

```bash
# Install dependencies
sudo pacman -S qt6-base qt6-pdf qt6-quick3d cmake gcc git

# Further steps same as Ubuntu
```

### Creating AppImage

```bash
# After building
cd build

# Install to separate directory
cmake --install . --prefix AppDir/usr

# Download linuxdeployqt
wget https://github.com/probonopd/linuxdeployqt/releases/download/continuous/linuxdeployqt-continuous-x86_64.AppImage
chmod +x linuxdeployqt-continuous-x86_64.AppImage

# Create AppImage
./linuxdeployqt-continuous-x86_64.AppImage AppDir/usr/bin/documentviewer -appimage
```

## 🍎 Building on macOS

```bash
# Install Xcode Command Line Tools
xcode-select --install

# Install Homebrew (if not already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install dependencies
brew install qt@6 cmake git

# Clone repository
git clone https://github.com/username/documentviewer.git
cd documentviewer

# Create build directory
mkdir build
cd build

# Configure project
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=$(brew --prefix qt@6)

# Build project
cmake --build . -j$(sysctl -n hw.ncpu)

# Run application
open app/documentviewer.app

# Create DMG (optional)
cpack -G DragNDrop
```

### Creating .app Bundle

```bash
# After building
cd build/app

# Deploy Qt dependencies
macdeployqt documentviewer.app -dmg

# Now you have documentviewer.dmg
```

## 🔧 CMake Options

### Basic Options

```bash
# Build type
-DCMAKE_BUILD_TYPE=Release          # Release, Debug, RelWithDebInfo, MinSizeRel

# Qt path
-DCMAKE_PREFIX_PATH=/path/to/Qt/6.8.0/compiler

# Installation prefix
-DCMAKE_INSTALL_PREFIX=/usr/local

# Build generator
-G "Visual Studio 17 2022"          # Windows
-G "Unix Makefiles"                 # Linux/macOS
-G "Ninja"                          # Fast builds
```

### Project Specific Options

Document Viewer automatically detects available Qt components:

```cmake
# Force disable PrintSupport
-DQt6PrintSupport_DIR=""

# Force disable PDF
-DQt6Pdf_DIR=""

# Force disable Quick3D
-DQt6Quick3D_DIR=""
```

### Example Configurations

```bash
# Minimal build (without PDF and 3D)
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DCMAKE_PREFIX_PATH=/path/to/Qt \
         -DQt6Pdf_DIR="" \
         -DQt6Quick3D_DIR=""

# Full build with all plugins
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DCMAKE_PREFIX_PATH=/path/to/Qt

# Debug build
cmake .. -DCMAKE_BUILD_TYPE=Debug \
         -DCMAKE_PREFIX_PATH=/path/to/Qt
```

## 🐛 Troubleshooting

### Qt Not Found

```bash
# Ensure CMAKE_PREFIX_PATH points to correct path
cmake .. -DCMAKE_PREFIX_PATH=/full/path/to/Qt/6.8.0/compiler
```

### C++17 Compiler Errors

```bash
# For older compilers explicitly specify standard
cmake .. -DCMAKE_CXX_STANDARD=17
```

### Plugins Don't Load

```bash
# Linux: check rpath
ldd ./app/documentviewer

# Ensure plugins are built
ls -la plugins/*/

# Check Qt finds plugins
export QT_DEBUG_PLUGINS=1
./app/documentviewer
```

### Path Issues on Windows

```cmd
:: Use full paths without spaces or
:: use short names
dir /x

:: Or use quotes
cmake .. -DCMAKE_PREFIX_PATH="C:\Program Files\Qt\6.8.0\msvc2022_64"
```

### PDF Plugin Build Errors

```bash
# Ensure Qt PDF is installed
# Ubuntu/Debian:
sudo apt install qt6-pdf-dev

# Or disable PDF plugin
cmake .. -DQt6Pdf_DIR=""
```

### Memory Issues During Build

```bash
# Limit parallel tasks
cmake --build . -j2  # Instead of -j$(nproc)
```

### Clean Build

```bash
# Remove build directory and recreate
rm -rf build
mkdir build
cd build
cmake ..
cmake --build .
```

## 📝 Additional Notes

### Environment Variables

```bash
# Linux/macOS
export CMAKE_PREFIX_PATH=/path/to/Qt/6.8.0/compiler
export PATH=$CMAKE_PREFIX_PATH/bin:$PATH
export LD_LIBRARY_PATH=$CMAKE_PREFIX_PATH/lib:$LD_LIBRARY_PATH

# Windows (PowerShell)
$env:CMAKE_PREFIX_PATH="C:\Qt\6.8.0\msvc2022_64"
$env:Path="$env:CMAKE_PREFIX_PATH\bin;$env:Path"
```

### Performance Recommendations

- Use **Ninja** generator for fast builds
- Enable **ccache** for faster rebuilds
- Use **-j** flag for parallel compilation

```bash
# With Ninja
cmake .. -G Ninja -DCMAKE_BUILD_TYPE=Release
ninja -j$(nproc)

# With ccache
cmake .. -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
```

## 🆘 Getting Help

If you encounter problems:

1. Check [Issues](../../issues) on GitHub
2. Create a new Issue with detailed description
3. Include:
   - Your OS and version
   - Qt version
   - CMake version
   - Compiler and version
   - Full error output
   - Commands you used

## 📚 Additional Resources

- [Qt Documentation](https://doc.qt.io/)
- [CMake Documentation](https://cmake.org/documentation/)
- [CONTRIBUTING.md](CONTRIBUTING.md) - for developers
- [README.md](README.md) - general project information
