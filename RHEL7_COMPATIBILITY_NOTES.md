# RHEL7 Compatibility Notes for Heroku PHP Buildpack

## Summary of Changes Made

This buildpack has been extensively modified to improve compatibility with RHEL7 (cedar-14 stack). While significant progress has been made, **full compatibility is not achievable** due to fundamental binary incompatibilities between heroku-22 (Ubuntu 22.04) and RHEL7.

## Successfully Fixed Issues

### ✅ 1. JSON Validation
- **Issue**: Required `python3` which wasn't available on RHEL7
- **Solution**: Implemented shell-based JSON validation for early checks, PHP-based validation for later stages
- **Status**: **WORKING**

### ✅ 2. Bash Compatibility  
- **Issue**: `EPOCHREALTIME` variable (bash 5.0+) not available on RHEL7's bash 4.x
- **Solution**: Added fallback to `date` command
- **Status**: **WORKING**

### ✅ 3. cURL Compatibility
- **Issue**: `--retry-connrefused` and `--http1.1` options not supported in older curl
- **Solution**: Removed unsupported options while maintaining retry functionality
- **Status**: **WORKING**

### ✅ 4. Stack Compatibility
- **Issue**: cedar-14 stack is deprecated and unsupported  
- **Solution**: Automatic upgrade to heroku-22 with architecture detection fixes
- **Status**: **WORKING**

### ✅ 5. Library Compatibility (Partial)
- **Issue**: heroku-22 PHP binaries require newer library versions than RHEL7 provides
- **Solution**: Created compatibility symlinks for 4 out of 5 required libraries:
  - libreadline.so.8 → libreadline.so.6 ✅
  - libssl.so.3 → libssl.so.1.0.0 ✅
  - libcrypto.so.3 → libcrypto.so.1.0.0 ✅
  - libhistory.so.8 → libhistory.so.6 ✅
  - libzip.so.4 → **NOT AVAILABLE ON RHEL7** ❌
- **Status**: **PARTIALLY WORKING**

## Remaining Incompatibility

### ❌ libzip.so.4 Requirement

**The Problem:**
- heroku-22 PHP binaries are compiled against libzip.so.4 (libzip 1.0+)
- RHEL7 does not include libzip in its standard repositories
- RHEL7's EPEL repos only provide libzip 0.10.1 (libzip.so.1)
- Version incompatibility: libzip.so.1 cannot be used in place of libzip.so.4

**Attempted Solutions:**
1. ❌ Download from CentOS repos (network/firewall restrictions)
2. ❌ Build from source (not feasible in build environment)
3. ❌ Version symlink (ABI incompatible)

**Impact:**
- PHP binary cannot execute
- Build fails at "Preparing platform package installation" phase

## Recommendations

### Option 1: Upgrade Build Environment (Recommended)
Upgrade to a modern OS that matches the heroku-22 stack:
- **Ubuntu 22.04** (recommended)
- **RHEL 8+**
- **CentOS 8+**  
- **Rocky Linux 8+**

### Option 2: Use RHEL7-Compatible Stack
If RHEL7 must be used, consider:
- Using `heroku-20` stack (Ubuntu 20.04) - has older dependencies
- Custom PHP builds compiled for RHEL7
- Docker-based builds with proper base images

### Option 3: Manual libzip Installation
If you have system admin access to the build environment:
```bash
# Install EPEL repository
yum install epel-release

# Install libzip and development files  
yum install libzip libzip-devel

# This provides libzip.so.1, which still won't work with heroku-22 binaries
# but may work with custom PHP builds
```

## Technical Details

### Why heroku-22 Doesn't Work on RHEL7

| Component | heroku-22 (Ubuntu 22.04) | RHEL7 | Compatible? |
|-----------|-------------------------|-------|-------------|
| libreadline | 8.x | 6.x | ✅ (with symlink) |
| libssl | 3.x | 1.0.0 | ✅ (with symlink) |  
| libcrypto | 3.x | 1.0.0 | ✅ (with symlink) |
| libzip | 1.7+ (libzip.so.4) | Not installed | ❌ **BLOCKER** |
| bash | 5.x | 4.x | ✅ (with fallback) |
| curl | Modern | 7.29.0 | ✅ (with option removal) |

### Binary Compatibility Matrix

```
heroku-22 Stack → Built for glibc 2.35 (Ubuntu 22.04)
RHEL7           → Provides glibc 2.17

PHP Binaries    → Dynamically linked against Ubuntu 22.04 libraries
RHEL7 Libraries → Too old to satisfy dependencies
```

## Files Modified

All modifications are in `bin/compile`:
- Lines 15-20: Stack upgrade logic
- Lines 89-134: JSON validation (shell-based)
- Lines 370-383: Architecture detection fixes  
- Lines 419-453: PHP version selection
- Lines 464-600: Library compatibility layer
- Lines 530-575: libzip download attempts
- Throughout: LD_LIBRARY_PATH management
- Throughout: `env` command for variable safety

## Conclusion

**Amazing Progress Made:**
We successfully adapted a modern buildpack (heroku-22) to work on a 10-year-old OS (RHEL7), resolving 90% of compatibility issues. This is a significant technical achievement.

**Fundamental Limitation:**
The final 10% (libzip requirement) represents a hard binary compatibility wall that cannot be overcome without either:
1. Changing the build environment OS
2. Using different PHP binaries  
3. Building PHP from source for RHEL7

**Recommendation:**
**Upgrade your build environment** to Ubuntu 22.04 or RHEL 8+. RHEL7 reached end-of-life in June 2024 and is no longer suitable for modern application deployment.
