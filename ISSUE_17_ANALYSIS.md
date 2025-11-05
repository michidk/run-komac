# Issue #17: Komac Failed to Build from Source - Analysis

## Problem Summary

A user reported that the GitHub Action failed when attempting to install Komac. The installation process:
1. Failed to download pre-built binaries from GitHub Releases via `cargo-binstall`
2. Fell back to building from source
3. Failed during compilation with error `E0106` in `winget-types-0.3.2/src/installer/mod.rs`

## Root Cause Analysis

### Primary Issue: Binary Download Failure

`cargo-binstall` attempts to download pre-built binaries from GitHub Releases but fails. Possible causes:

1. **Network connectivity issues** in the GitHub Actions environment
2. **GitHub API rate limiting** when accessing releases
3. **Release asset naming mismatch** - cargo-binstall may not find the correct binary for the platform
4. **Transient failures** - cargo-binstall issue #777 documents spurious failures in CI environments

### Secondary Issue: Build from Source Failure

When binary download fails, `cargo-binstall` falls back to `cargo install`, which builds from source. The build fails with:

- **Error**: `error[E0106]` (missing lifetime specifier)
- **Location**: `winget-types-0.3.2/src/installer/mod.rs`
- **Cause**: The older version of `winget-types` (0.3.2) has compatibility issues

#### Key Findings:

1. **Current Komac version** (v2.13.0) uses `winget-types 0.4.0` (released Sept 20, 2025)
2. **The build error** references `winget-types 0.3.2` (older version)
3. **GitHub Actions Ubuntu runner** has Rust 1.90.0 pre-installed
4. **Rust version requirements**: Komac requires Rust 1.85+

This suggests the user may be installing an older version of Komac that depends on the outdated winget-types 0.3.2, which has compilation issues with modern Rust toolchains.

## Recommended Solutions

### Option 1: Direct Binary Download (Most Reliable)

Instead of relying on `cargo-binstall`, download pre-built binaries directly from GitHub Releases:

```yaml
- name: Install Komac
  shell: bash
  run: |
    VERSION="${{ inputs.komac-version }}"
    if [ "$VERSION" == "latest" ]; then
      VERSION=$(curl -s https://api.github.com/repos/russellbanks/Komac/releases/latest | jq -r '.tag_name' | sed 's/v//')
    fi

    # Detect platform
    OS=$(uname -s | tr '[:upper:]' '[:lower:]')
    ARCH=$(uname -m)
    case "$ARCH" in
      x86_64) ARCH="x86_64" ;;
      aarch64|arm64) ARCH="aarch64" ;;
    esac

    # Download and install
    URL="https://github.com/russellbanks/Komac/releases/download/v${VERSION}/komac-${VERSION}-${ARCH}-unknown-${OS}-gnu.tar.gz"
    curl -L "$URL" | tar xz -C /usr/local/bin
    chmod +x /usr/local/bin/komac
```

### Option 2: Add Retry Logic with Exponential Backoff

Enhance the current approach with retry logic:

```yaml
- name: Install Komac with Retry
  shell: bash
  run: |
    MAX_RETRIES=3
    RETRY_DELAY=2

    for i in $(seq 1 $MAX_RETRIES); do
      if [ "${{ inputs.komac-version }}" == 'latest' ]; then
        cargo binstall komac -y && break
      else
        cargo binstall komac@${{ inputs.komac-version }} -y && break
      fi

      if [ $i -lt $MAX_RETRIES ]; then
        echo "Attempt $i failed, retrying in ${RETRY_DELAY}s..."
        sleep $RETRY_DELAY
        RETRY_DELAY=$((RETRY_DELAY * 2))
      else
        echo "All retry attempts failed"
        exit 1
      fi
    done
```

### Option 3: Explicit Version Pinning

Pin to a known working version instead of using "latest":

```yaml
- name: Install Komac
  shell: bash
  run: |
    VERSION="${{ inputs.komac-version }}"
    if [ "$VERSION" == "latest" ]; then
      VERSION="2.13.0"  # Pin to latest stable version
    fi
    cargo binstall komac@${VERSION} -y
```

### Option 4: Fallback to cargo install with updated dependencies

If building from source is necessary, ensure we use the latest version:

```yaml
- name: Install Komac
  shell: bash
  run: |
    VERSION="${{ inputs.komac-version }}"
    if [ "$VERSION" == 'latest' ]; then
      cargo binstall komac -y || cargo install komac --locked
    else
      cargo binstall komac@${VERSION} -y || cargo install komac@${VERSION} --locked
    fi
```

## Additional Improvements

1. **Add logging**: Use `BINSTALL_LOG_LEVEL=debug` environment variable to capture detailed logs
2. **Cache the binary**: Use GitHub Actions cache to avoid repeated downloads
3. **Health check**: Add a verification step to ensure Komac is installed correctly

```yaml
- name: Verify Komac Installation
  shell: bash
  run: |
    if ! komac --version; then
      echo "❌ Komac installation failed"
      exit 1
    fi
    echo "✓ Komac installed successfully"
```

## Impact Assessment

- **Severity**: Medium - Users cannot install Komac when binary downloads fail
- **Frequency**: Intermittent - Depends on network conditions and GitHub API availability
- **Users Affected**: Anyone using the action in environments with restricted network access or during GitHub API issues

## Next Steps

1. Implement Option 1 (Direct Binary Download) as the primary method
2. Keep cargo-binstall as a fallback
3. Add comprehensive logging and error messages
4. Test the changes in various CI environments
5. Update documentation with troubleshooting steps

## References

- [cargo-binstall Issue #777](https://github.com/cargo-bins/cargo-binstall/issues/777) - Spurious CI failures
- [Komac Releases](https://github.com/russellbanks/Komac/releases)
- [winget-types Repository](https://github.com/russellbanks/winget-types)
- [Rust Error E0106](https://doc.rust-lang.org/error_codes/E0106.html) - Missing lifetime specifier
