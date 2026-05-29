# Laboratory work IV

# Homework

В корневой CMakeLists.txt добавлены эти строчки перед project:

```cmake
set(PRINT_VERSION_MAJOR 0)
set(PRINT_VERSION_MINOR 1)
set(PRINT_VERSION_PATCH 0)
set(PRINT_VERSION_TWEAK 0)
set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
set(PRINT_VERSION_STRING "v${PRINT_VERSION}")
```

И в конец:
```cmake
include(CPackConfig.cmake)
```

```bash
$ touch CPackConfig.cmake
$ nano CPackConfig.cmake
```

В CPackConfig.cmake вставлено:
```cmake
include(InstallRequiredSystemLibraries)

set(CPACK_PACKAGE_CONTACT ${GITHUB_EMAIL})
set(CPACK_PACKAGE_VERSION_MAJOR ${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PRINT_VERSION})

set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/DESCRIPTION")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Solver application for mathematical problems")

set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/README.md")

set(CPACK_DEBIAN_PACKAGE_NAME "libsolver-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")

set(CPACK_RPM_PACKAGE_NAME "solver-devel")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "Development/Tools")

include(CPack)
```

Дальше изменяем build.yml:
```yaml
name: CI

on:
  push:
    branches: [ "main", "master" ]
    tags:
      - 'v*'
  pull_request:
    branches: [ "main", "master" ]

jobs:
  build:
    name: ${{ matrix.os }} / ${{ matrix.compiler }}
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            compiler: gcc
            cc: gcc
            cxx: g++

          - os: ubuntu-latest
            compiler: clang
            cc: clang
            cxx: clang++

          - os: windows-latest
            compiler: msvc

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Linux compilers
        if: runner.os == 'Linux'
        run: sudo apt-get update && sudo apt-get install -y build-essential clang

      - name: Setup MSVC environment
        if: runner.os == 'Windows'
        uses: ilammy/msvc-dev-cmd@v1

      - name: Configure CMake (Unix)
        if: runner.os != 'Windows'
        run: cmake -S . -B _build -DCMAKE_INSTALL_PREFIX=_install -DCMAKE_C_COMPILER=${{ matrix.cc }} -DCMAKE_CXX_COMPILER=${{ matrix.cxx }}

      - name: Configure CMake (Windows)
        if: runner.os == 'Windows'
        run: cmake -S . -B _build -DCMAKE_INSTALL_PREFIX=_install

      - name: Build (Unix)
        if: runner.os != 'Windows'
        run: cmake --build _build

      - name: Build (Windows)
        if: runner.os == 'Windows'
        run: cmake --build _build --config Release

  release-packages:
    name: Build packages
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            cpack_generator: DEB;RPM
            extra_packages: rpm
          - os: windows-latest
            cpack_generator: WIX
            extra_packages: wixtoolset
          - os: macos-latest
            cpack_generator: DragNDrop
            extra_packages: ""

    steps:
      - uses: actions/checkout@v4

      - name: Install Linux packaging tools
        if: runner.os == 'Linux'
        run: sudo apt-get update && sudo apt-get install -y rpm

      - name: Install WiX Toolset (Windows)
        if: runner.os == 'Windows'
        run: choco install wixtoolset

      - name: Configure CMake
        run: cmake -S . -B _build -DCMAKE_BUILD_TYPE=Release

      - name: Build
        run: cmake --build _build --config Release

      - name: Create packages with CPack
        shell: bash
        run: |
          cd _build
          IFS=';' read -ra GENERATORS <<< "${{ matrix.cpack_generator }}"
          for gen in "${GENERATORS[@]}"; do
            echo "=== Generating $gen ==="
            cpack -G "$gen"
          done

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: packages-${{ matrix.os }}
          path: _build/*.{deb,rpm,dmg,msi}

  create-release:
    needs: release-packages
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: write
    steps:
      - name: Download all package artifacts
        uses: actions/download-artifact@v4
        with:
          path: artifacts

      - name: Create Release and upload assets
        uses: softprops/action-gh-release@v2
        with:
          files: artifacts/**/*.{deb,rpm,dmg,msi}
          generate_release_notes: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Созданы файлы для корректной работы
```bash
$ touch DESCRIPTION LICENSE ChangeLog.md
```

```bash
$ git add .github/workflows/build.yml CMakeLists.txt CPackConfig.cmake DESCRIPTION
$ git commit -m "Add CPack"
$ git push origin main
$ git tag v1.0.0
$ git push origin --tags
```

Файл собрался, всё работает.
