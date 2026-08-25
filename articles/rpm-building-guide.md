# RPM Building Guide

A practical guide to building RPM packages on Red Hat Enterprise Linux. Covers the rpmbuild directory structure, spec file anatomy, build stages, and the complete flow from source tarball to installable RPM.

---

## Build Environment Setup

```bash
# Install build tools
dnf install rpm-build rpmdevtools gcc make

# Create the rpmbuild directory tree
rpmdev-setuptree

# Or manually
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
```

### Directory Structure

```
%_topdir/rpmbuild/
├── SOURCES/          ← Source tarballs, patches, additional files
├── SPECS/            ← Spec files (.spec) defining how to build
├── BUILD/            ← Unpacked sources and compilation work area
├── BUILDROOT/        ← Fake install root ($RPM_BUILD_ROOT staging area)
├── RPMS/             ← Output: binary RPMs (by architecture)
│   └── x86_64/
│       └── mypackage-1.0-1.el9.x86_64.rpm
└── SRPMS/            ← Output: source RPMs
    └── mypackage-1.0-1.el9.src.rpm
```

| Directory | Purpose |
|-----------|---------|
| `SOURCES` | Contains source tarballs (`.tar.gz`), patches (`.patch`), and config files |
| `SPECS` | Contains the `.spec` file that defines the entire build process |
| `BUILD` | Working directory where sources are unpacked and compiled |
| `BUILDROOT` | Staging area simulating the target filesystem (`$RPM_BUILD_ROOT`) |
| `RPMS` | Final binary RPMs, organized by architecture (`x86_64`, `noarch`) |
| `SRPMS` | Final source RPMs (complete package source for rebuilding) |

---

## RPM Build Flow

The `rpmbuild` command processes a spec file through several stages. Each stage can be invoked individually for debugging:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        rpmbuild Flow                                │
│                                                                     │
│  ❶ %prep     → Unpack SOURCES tarball into BUILD/                   │
│               (runs %setup / %patch macros)                         │
│                                                                     │
│  ❷ %build    → Compile source code in BUILD/                        │
│               (runs ./configure && make)                            │
│                                                                     │
│  ❸ %install  → Install files into BUILDROOT/ (staging area)         │
│               (runs make install DESTDIR=%{buildroot})              │
│                                                                     │
│  ❹ %check    → Run test suite (optional)                            │
│               (runs make test / make check)                         │
│                                                                     │
│  ❺ Package   → Create binary RPM from BUILDROOT/ → RPMS/            │
│               (files listed in %files section)                      │
│                                                                     │
│  ❻ Source    → Create source RPM from SOURCES/ + SPECS/ → SRPMS/    │
│               (complete reproducible build)                         │
└─────────────────────────────────────────────────────────────────────┘
```

### rpmbuild Stage Flags

| Flag | Stage | What It Does |
|------|-------|--------------|
| `-bp` | %prep only | Unpack sources and apply patches |
| `-bc` | %prep + %build | Compile the source |
| `-bi` | %prep + %build + %install | Install into BUILDROOT |
| `-bl` | File list check | Verify %files section matches BUILDROOT |
| `-bb` | Binary RPM | Complete build → produce binary RPM |
| `-ba` | All (binary + source) | Produce both binary and source RPMs |
| `-bs` | Source RPM only | Produce only the SRPM |

```bash
# Build everything (most common)
rpmbuild -ba ~/rpmbuild/SPECS/mypackage.spec

# Just prep (useful for debugging spec issues)
rpmbuild -bp ~/rpmbuild/SPECS/mypackage.spec

# Build binary RPM only (no SRPM)
rpmbuild -bb ~/rpmbuild/SPECS/mypackage.spec

# Build source RPM only
rpmbuild -bs ~/rpmbuild/SPECS/mypackage.spec
```

---

## Spec File Anatomy

```bash
# ~/rpmbuild/SPECS/mypackage.spec

# === Header Section ===
Name:           mypackage
Version:        1.0
Release:        1%{?dist}
Summary:        My example package
License:        GPLv2+
URL:            https://example.com/mypackage
Source0:        %{name}-%{version}.tar.gz
Patch0:         fix-typo.patch

BuildRequires:  gcc, make
Requires:       bash

%description
This is an example package that demonstrates RPM building.

# === Prep Section ===
%prep
%setup -q
%patch0 -p1

# === Build Section ===
%build
%configure
make %{?_smp_mflags}

# === Install Section ===
%install
rm -rf %{buildroot}
make install DESTDIR=%{buildroot}

# === Check Section (optional) ===
%check
make test

# === Files Section ===
%files
%license LICENSE
%doc README.md
%{_bindir}/mycommand
%{_mandir}/man1/mycommand.1*
%config(noreplace) %{_sysconfdir}/mypackage.conf

# === Changelog Section ===
%changelog
* Mon Aug 25 2026 Your Name <you@example.com> - 1.0-1
- Initial package
```

### Key Spec File Sections

| Section | Purpose |
|---------|---------|
| Header | Package metadata — name, version, dependencies, sources |
| `%description` | Multi-line package description |
| `%prep` | Unpack sources, apply patches |
| `%build` | Compile (configure, make) |
| `%install` | Install to `%{buildroot}` (fake root) |
| `%check` | Run tests (optional but recommended) |
| `%files` | List of files to include in the RPM |
| `%changelog` | Version history in RPM changelog format |

### Important Macros

| Macro | Expands To | Example |
|-------|-----------|---------|
| `%{name}` | Package name | `mypackage` |
| `%{version}` | Package version | `1.0` |
| `%{release}` | Release number | `1.el9` |
| `%{?dist}` | Distribution tag | `.el9` |
| `%{buildroot}` | BUILDROOT path | `~/rpmbuild/BUILDROOT/mypackage-1.0-1.el9.x86_64` |
| `%{_bindir}` | `/usr/bin` | |
| `%{_sbindir}` | `/usr/sbin` | |
| `%{_sysconfdir}` | `/etc` | |
| `%{_libdir}` | `/usr/lib64` (on 64-bit) | |
| `%{_datadir}` | `/usr/share` | |
| `%{_mandir}` | `/usr/share/man` | |
| `%{_unitdir}` | `/usr/lib/systemd/system` | |
| `%{?_smp_mflags}` | `-jN` (parallel make) | `-j8` |

```bash
# See all available macros
rpm --showrc | less

# Query a specific macro
rpm -E '%{_bindir}'
rpm -E '%{?dist}'
```

---

## Building from Source Tarball — Complete Example

### 1. Prepare the Source

```bash
# Create source directory and tarball
mkdir mypackage-1.0
echo '#!/bin/bash' > mypackage-1.0/mycommand
echo 'echo "Hello from mypackage"' >> mypackage-1.0/mycommand
chmod +x mypackage-1.0/mycommand

# Create the tarball
tar czf ~/rpmbuild/SOURCES/mypackage-1.0.tar.gz mypackage-1.0/
```

### 2. Write the Spec File

```bash
cat > ~/rpmbuild/SPECS/mypackage.spec << 'EOF'
Name:           mypackage
Version:        1.0
Release:        1%{?dist}
Summary:        A simple example package
License:        GPLv2+
Source0:        %{name}-%{version}.tar.gz
BuildArch:      noarch

%description
A simple shell script packaged as an RPM example.

%prep
%setup -q

%install
mkdir -p %{buildroot}%{_bindir}
install -m 755 mycommand %{buildroot}%{_bindir}/mycommand

%files
%{_bindir}/mycommand

%changelog
* Mon Aug 25 2026 Admin <admin@example.com> - 1.0-1
- Initial package
EOF
```

### 3. Build the RPM

```bash
rpmbuild -ba ~/rpmbuild/SPECS/mypackage.spec
```

### 4. Verify the Output

```bash
# List produced RPMs
ls ~/rpmbuild/RPMS/noarch/
# mypackage-1.0-1.el9.noarch.rpm

ls ~/rpmbuild/SRPMS/
# mypackage-1.0-1.el9.src.rpm

# Inspect the RPM
rpm -qpi ~/rpmbuild/RPMS/noarch/mypackage-1.0-1.el9.noarch.rpm
rpm -qpl ~/rpmbuild/RPMS/noarch/mypackage-1.0-1.el9.noarch.rpm

# Install and test
sudo rpm -ivh ~/rpmbuild/RPMS/noarch/mypackage-1.0-1.el9.noarch.rpm
mycommand
```

---

## Rebuilding from SRPM

```bash
# Install SRPM (populates ~/rpmbuild/SOURCES and ~/rpmbuild/SPECS)
rpm -ivh mypackage-1.0-1.el9.src.rpm

# Rebuild
rpmbuild -ba ~/rpmbuild/SPECS/mypackage.spec

# Or rebuild directly from SRPM
rpmbuild --rebuild mypackage-1.0-1.el9.src.rpm
```

---

## Common Patterns

### Packaging a Config File

```bash
%install
mkdir -p %{buildroot}%{_sysconfdir}
install -m 644 mypackage.conf %{buildroot}%{_sysconfdir}/mypackage.conf

%files
%config(noreplace) %{_sysconfdir}/mypackage.conf
```

`%config(noreplace)` means: on upgrade, if the user modified the file, keep their version and save the new one as `.rpmnew`.

### Packaging a systemd Service

```bash
%install
mkdir -p %{buildroot}%{_unitdir}
install -m 644 mypackage.service %{buildroot}%{_unitdir}/mypackage.service

%post
%systemd_post mypackage.service

%preun
%systemd_preun mypackage.service

%postun
%systemd_postun_with_restart mypackage.service

%files
%{_unitdir}/mypackage.service
```

### Packaging Documentation

```bash
%files
%license LICENSE
%doc README.md CHANGELOG.md
%doc %{_mandir}/man1/mycommand.1*
```

### Sub-packages

```bash
%package devel
Summary:  Development files for %{name}
Requires: %{name}%{?_isa} = %{version}-%{release}

%description devel
Header files and libraries for developing with %{name}.

%files devel
%{_includedir}/%{name}/
%{_libdir}/lib%{name}.so
```

---

## Debugging Build Failures

```bash
# Prep only — check if sources unpack correctly
rpmbuild -bp ~/rpmbuild/SPECS/mypackage.spec

# Build only — check compilation
rpmbuild -bc ~/rpmbuild/SPECS/mypackage.spec

# Install only — check BUILDROOT layout
rpmbuild -bi ~/rpmbuild/SPECS/mypackage.spec

# Check file list — find missing/unpackaged files
rpmbuild -bl ~/rpmbuild/SPECS/mypackage.spec

# Show what the BUILDROOT contains
find ~/rpmbuild/BUILDROOT/ -type f

# Verbose build with debug output
rpmbuild -ba --verbose ~/rpmbuild/SPECS/mypackage.spec
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Installed (but unpackaged) file(s) found` | Files in BUILDROOT not listed in %files | Add to %files or exclude with `%exclude` |
| `File not found` in %files | File path doesn't match BUILDROOT layout | Check `%install` creates correct paths |
| `Bad exit status from %prep` | Tarball structure doesn't match `%setup -q` | Verify tarball extracts to `%{name}-%{version}/` |
| `No package provides X` (BuildRequires) | Missing build dependency | `dnf install` the dependency |
| `Permission denied` | Building as root | Always build as non-root user |

---

## RPM Query Commands

```bash
# Query installed package info
rpm -qi mypackage

# List files in installed package
rpm -ql mypackage

# List files in an RPM file (not installed)
rpm -qpl mypackage-1.0-1.el9.noarch.rpm

# Show package info from RPM file
rpm -qpi mypackage-1.0-1.el9.noarch.rpm

# Find which package owns a file
rpm -qf /usr/bin/mycommand

# Show changelog
rpm -q --changelog mypackage

# Verify installed package (check file integrity)
rpm -V mypackage

# Show dependencies
rpm -qR mypackage

# Show scripts (pre/post install)
rpm -q --scripts mypackage
```

---

## Quick Reference

```bash
# === Setup ===
rpmdev-setuptree
# Creates ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

# === Build commands ===
rpmbuild -ba package.spec    # Build binary + source RPM
rpmbuild -bb package.spec    # Build binary RPM only
rpmbuild -bs package.spec    # Build source RPM only
rpmbuild -bp package.spec    # Prep only (unpack)
rpmbuild -bc package.spec    # Prep + compile
rpmbuild -bi package.spec    # Prep + compile + install
rpmbuild -bl package.spec    # Check file list
rpmbuild --rebuild pkg.src.rpm  # Rebuild from SRPM

# === Macros ===
rpm -E '%{_bindir}'          # Show macro expansion
rpm --showrc                 # Show all macros

# === Query ===
rpm -qi package              # Info
rpm -ql package              # File list
rpm -qf /path/to/file       # Which package owns file
rpm -qpl file.rpm            # Files in RPM (not installed)
rpm -V package               # Verify integrity
```
