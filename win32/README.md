# Win32 ClamAV Build Instructions

This document describes how to build ClamAV on Windows using Visual Studio.
For information on how to use ClamAV, please refer to our [User Manual](https://www.clamav.net/documents).

## News

### Installer Projects

ClamAV 0.101 removed the old Visual Studio Installer Projects files (Setup-x64.vdproj, Setup-x86.vdproj). In their place we now build an installer using Inno Setup that is capable of installing ClamAV on both 32-bit and 64-bit architectures with one installer.

For more details, see the instructions below on how to build ClamAV.

### External library dependencies.

ClamAV relies on a handful of 3rd party libraries. In previous versions of ClamAV, most of these were copy-pasted into the win32/3rdparty directory, with the exception being OpenSSL. In ClamAV 0.102, all of these libraries are now external to ClamAV and must be compiled ahead of time as DLLs (or for zlib, a static lib) and placed in the %CLAM_DEPENDENCIES% (typically C:\clam_dependencies) directory so the ClamAV Visual Studio project files can find them. 

To build each of these libraries, we recommend using [Mussels](https://github.com/Cisco-Talos/mussels).  Mussels is an open source application dependency build tool that can build the correct version of each dependency using the build tools intended by the original library authors.

### Socket and libclamav API input

Starting from version 0.98 the windows version of ClamAV requires all the input to be UTF-8 encoded.

This affects:

- the API, notably the cl_scanfile() function
- clamd socket input, e.g. the commands SCAN, CONTSCAN, MUTLISCAN, etc.
- clamd socket output, i.e replies to the above queries

For legacy reasons ANSI (i.e. CP_ACP) input will still be accepted and processed as before, but with two important remarks:
First, socket replies to ANSI queries will still be UTF-8 encoded.
Second, ANSI sequences which are also valid UTF-8 sequences will be handled as UTF-8.

As a side note, console output (stdin and stderr) will always be OEM encoded,
even when redirected to a file.

## Requirements

To build the source code you will need:

- [Git for Windows](https://git-scm.com/download/win "Git SCM Windows Downloads") with a git "shell"
- [Microsoft Visual Studio 2017](https://www.visualstudio.com/vs/older-downloads/ "Visual Studio Downloads"): the community version is just fine.

To build the installer, you also need:

- [Inno Setup 5](http://www.jrsoftware.org/isdl.php "Inno Setup installer creation tool")

ClamAV is supported for Windows 7+, but Windows 10 is recommended.
Visual Studio 2019 should work fine, provided you install the platform toolkit for 2017.

## Getting the code

ClamAV source code is freely available on [GitHub](https://github.com/Cisco-Talos/clamav-devel)

To obtain a copy of the code, open a Git Bash terminal. Navigate to a directory where you want to store the code, eg "workspace" and clone the repository using the https web URL.  For example:
```cmd
cd
mkdir workspace
cd workspace
git clone https://github.com/vrtadmin/clamav-devel.git
```

Step into the win32 directory and open an Explorer window.
```cmd
cd clamav-devel
cd win32
explorer .
```

ClamAV for Windows uses the same code base as Unix/Linux based operating systems. However, Windows specific files for building ClamAV are found under the win32 directory.

## Code configuration

After downloading the source code, minimal configuration is required:

1. Run the `win32/configure.bat` script *from within the git shell*. Skip this step if you are building from an official release tarball.
2. Build the clamav_deps recipe collection using [Mussels](https://github.com/Cisco-Talos/mussels).  Mussels will install the headers and binaries in a directory structure similar to the tree described below, which is the format required by ClamAV Visual Studio project files and the `ClamAV-Installer.iss` InnoSetup 5 script.
    ```
    C:\clam_dependencies
    │
    ├───vcredist
    │   ├───vc_redist.x64.exe <-- VS 2017 Redistributables installer (x64)
    │   └───vc_redist.x86.exe <-- VS 2017 Redistributables installer (x86)
    ├───Win32
    │   ├───include
    │   │   └───openssl  <-- openssl headers here
    │   └───lib          <-- .DLLs and .LIBs here
    └───x64
        ├───include
        │   └───openssl  <-- openssl headers here
        └───lib          <-- .DLLs and .LIBs here
    ```
3. Copy `mussels\out\install` to `C:\clam_dependencies`*. Rename `C:\clam_dependencies\x86` to `C:\clam_dependencies\Win32`.
4. Add an environment variable with the name `CLAM_DEPENDENCIES` and set the value to the path of the above directory. 

   *Note: At present, the Inno Setup script `ClamAV-Installer.iss` requires this directory to be located specifically at `C:\clam_dependencies` in order to build the installer.

## Compilation

Open `win32/ClamAV.sln` in Visual Studio and build all.
The output directory for the binaries is either `/win32/(Win32|x64)/Debug` or
`/win32/(Win32|x64)/Release` depending on the configuration you pick.

Alternatively, you can build from the command line (aka `cmd.exe`) by following these steps:

x64:
```cmd
call "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Auxiliary\\Build\\vcvarsall.bat" x64
setx CLAM_DEPENDENCIES "C:\\clam_dependencies"
call configure.bat
devenv ClamAV.sln /Clean "Release|x64" /useenv /ProjectConfig "Release|x64"
devenv ClamAV.sln /Rebuild "Release|x64" /useenv /ProjectConfig "Release|x64"'''
```

x86:
```cmd
reg Query "HKLM\\Hardware\\Description\\System\\CentralProcessor\\0" | find /i "x86" > NUL && set OS=32BIT || set OS=64BIT
if %OS%==32BIT call "C:\\Program Files\\Microsoft Visual Studio\\2017\\Community\\VC\\Auxiliary\\Build\\vcvarsall.bat" x86
if %OS%==64BIT call "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Auxiliary\\Build\\vcvarsall.bat" x86
setx CLAM_DEPENDENCIES "C:\\clam_dependencies"
call configure.bat
devenv ClamAV.sln /Clean "Release|Win32" /useenv /ProjectConfig "Release|Win32"
devenv ClamAV.sln /Rebuild "Release|Win32" /useenv /ProjectConfig "Release|Win32"'''
```

To build the installer:

1. Build ClamAV for both `x64` **and** `Win32`. The installer requires both versions to be available.
2. Open `win32\ClamAV-Installer.iss` using Inno Setup 5.  
3. Run "Compile".

Alternatively, you can invoke the Inno Setup command line installer from cmd.exe:

```cmd
"C:\Program Files (x86)\Inno Setup 5\ISCC.exe" .\ClamAV-Installer.iss
```

After compilation, the installer will be located at `win32\ClamAV-<version>.exe`

## Special notes

The ClamAV tools in `win32` are the same as in unix, so refer to their respective
manpage for general usage.
The major differences are listed below:

- Config files path search order:
  1. The content of the registry key:
     "HKEY_LOCAL_MACHINE/Software/ClamAV/ConfDir"
  2. The directory where libclamav.dll is located:
     "C:\Program Files\ClamAV"
  3. "C:\ClamAV"

- Database files path search order:
  1. The content of the registry key:
     "HKEY_LOCAL_MACHINE/Software/ClamAV/DataDir"
  2. The directory "database" inside the directory where libclamav.dll is located:
     "C:\Program Files\ClamAV\database"
  3. "C:\ClamAV\db"

- Globbing
Since the Windows command prompt doesn't take care of wildcard expansion,
minimal emulation of unix glob() is performed internally.
It supports "*" and "?" only.

- File paths
Please always use the backslash as the path separator.
SMB Network shares and UNC paths are supported.

- Debug builds
Malloc in Debug (as opposed to release) mode fails after allocating some 90k
chunks; such builds won't be able to handle large databases.
Just do yourself a favour and always build in Release mode.

Special thanks
--------------

Special thanks to Gianluigi Tiesi and Mark Pizzolato for their valuable help in
coding and testing.
