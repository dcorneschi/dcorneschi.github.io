# Terraform Lock File Checksums: zh and h1 Hashes

## Overview

The `.terraform.lock.hcl` file stores cryptographic hashes that verify the integrity of downloaded providers. Two hash schemes exist: `zh:` (a legacy scheme tied to zip archives) and `h1:` (a directory hash of the unpacked provider). Understanding how these are calculated helps when debugging lock file mismatches or verifying providers manually.


## zh and h1 Checksums

The `terraform init` command downloads the provider and verifies that it matches one of the `zh:` checksums in the lock file, then extracts the package into `.terraform/providers`.

It also calculates a new `h1:` hash for that package, because Terraform has a "hash scheme upgrade" mechanism where it considers it safe to calculate a hash of a newer scheme if the package matches at least one hash of a legacy scheme. (`zh:` is a legacy scheme, usable only for `.zip` packages retrieved via the main registry protocol.)

- The `zh` is simply a SHA256 hash of the zip file which contains a provider for a specific OS/hardware platform combination.
- The `h1` hash is a so-called `dirhash` of the provider's directory (unzipped).

The dirhash (`h1`) is created from the list of `sha256sum` filenames. Once this list is sha256sum'd again, the resulting hash is taken in binary representation and then converted to Base64.


## Calculate h1 Hash

Navigate to the extracted provider directory and compute the hash step by step:

```sh
cd .terraform/providers/registry.terraform.io/bpg/proxmox/0.69.0/linux_amd64

ll
# total 22316
# -rw-r--r--. 1 root root   212961 Mar  5 10:07 CHANGELOG.md
# -rw-r--r--. 1 root root    16725 Mar  5 10:07 LICENSE
# -rw-r--r--. 1 root root     7218 Mar  5 10:07 README.md
# -rwxr-xr-x. 1 root root 22605976 Mar  5 10:07 terraform-provider-proxmox_v0.69.0
```

Generate the sha256 sums of all files, then hash that list:

```sh
sha256sum * > /tmp/dirhash

sha256sum /tmp/dirhash
# 50b50edc05097e12f10d4d3692d560552db8307d57caf2a41c0ba3b3f2982352  /tmp/dirhash
```

Convert the hex hash to binary and then Base64-encode it:

```sh
echo 50b50edc05097e12f10d4d3692d560552db8307d57caf2a41c0ba3b3f2982352 \
  | ruby -rbase64 -e 'puts Base64.encode64 [STDIN.read.chomp].pack("H*")'
# ULUO3AUJfhLxDU02ktVgVS24MH1XyvKkHAujs/KYI1I=
```

This matches the `h1:` entry in the lock file:

```hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/bpg/proxmox" {
  version     = "0.69.0"
  constraints = "0.69.0"
  hashes = [
    "h1:ULUO3AUJfhLxDU02ktVgVS24MH1XyvKkHAujs/KYI1I=",
    "zh:046713ab723f4aecc2886263b3e2fc79f2391c821a81a5346f7ff185edd17f68",
    "zh:05c19166978a8a81031e502d3934bae5daac17fe44d8f397bb6a67f9bade337b",
    "zh:12327ed39e85680cfd086bcb0d7ebefd15d352c1cd857e5164d4729122821489",
    "zh:4f833932192a136dbafc54ee98dcfeb612dc7b679ba5bcb59f7d430721b58f80",
    "zh:6c5547ee42a6ed6ae40a707c97fd1bf22b082feed8d31f34bcc9447018b7a2c5",
    "zh:6ee9fe5d73fe283cc4c6cb551b7a5ccd857be65f91872446b772f75f75a2a272",
    "zh:8a4d23aa38298286bee221db01a8f02492679e5ab877eaa793df4f16af4ed714",
    "zh:982011abf6ce4499d6b8e00aa7d7ba92229ae641fa8e631b14ced37343f443cc",
    "zh:a46683898b8d193f40de3837c6ea2bbf8a68ac59e6d4463c307a9931cccb5e42",
    "zh:ce3ea79bd1b4f3d881e7de8d2e9e0bf86f0c48ad1b71ff4ce48f0ba09b732106",
    "zh:d20d861810452ee57670d0389e8409644f7b61888c8c9cc67f65cdb06fc3456d",
    "zh:d6169bdacfc2f88decf2c8f3af47bbf411de914120e128cd53af639a707b6d13",
    "zh:e8690a35444bfdd3899fef16afcce1ccf4ab9b7140f53e23ba96aa623f84e6c5",
    "zh:f26e0763dbe6a6b2195c94b44696f2110f7f55433dc142839be16b9697fa5597",
    "zh:f9c0df46f852e241eb6342d684466dd9de4b8a1058f1453fbe1ec0ffb6d1fe1a",
  ]
}
```


## How h1 Actually Works (Under the Hood)

Terraform's `h1:` hash is implemented using Go's [`golang.org/x/mod/sumdb/dirhash`](https://github.com/golang/mod/blob/master/sumdb/dirhash/hash.go) package — the same algorithm used by Go modules in `go.sum` files. Terraform calls `dirhash.HashDir()` for unpacked directories and `dirhash.HashZip()` for zip archives.

The algorithm (`Hash1`) works as follows:

1. List all files in the directory recursively
2. Sort the file list alphabetically
3. For each file, compute SHA256 of its content
4. Write a line in the format: `<hex-sha256>  <filename>\n` (two spaces separator)
5. SHA256 the entire concatenated output from step 4
6. Base64-encode the final hash
7. Prefix with `h1:`

The Go implementation (simplified):

```go
func Hash1(files []string, open func(string) (io.ReadCloser, error)) (string, error) {
    h := sha256.New()
    files = append([]string(nil), files...)
    slices.Sort(files)
    for _, file := range files {
        r, _ := open(file)
        hf := sha256.New()
        io.Copy(hf, r)
        r.Close()
        fmt.Fprintf(h, "%x  %s\n", hf.Sum(nil), file)
    }
    return "h1:" + base64.StdEncoding.EncodeToString(h.Sum(nil)), nil
}
```

Key details:
- The hash covers **file paths and file contents** only — not permissions, timestamps, or other metadata
- File names are sorted using Go's default string sorting (lexicographic, byte-wise)
- The line format uses **two spaces** between hash and filename (matching `sha256sum` output format)
- An empty prefix is passed for provider directories, so file names are relative paths without a leading prefix

### How zh Works

The `zh:` scheme is simpler — it's a plain SHA256 of the `.zip` file:

```go
func PackageHashLegacyZipSHA(loc PackageLocalArchive) (Hash, error) {
    f, _ := os.Open(archivePath)
    defer f.Close()
    h := sha256.New()
    io.Copy(h, f)
    return HashSchemeZip.New(fmt.Sprintf("%x", h.Sum(nil))), nil
}
```

The result is `zh:` followed by the lowercase hex-encoded SHA256, which is designed to exactly match the format used in the registry API's `SHA256SUMS` file.

### Hash Verification Logic

When Terraform verifies a package, it uses `PackageMatchesAnyHash` which:

1. Iterates through all hashes in the lock file for that provider
2. For `h1:` hashes — computes `dirhash.HashDir()` of the unpacked directory (cached after first computation)
3. For `zh:` hashes — computes SHA256 of the `.zip` archive (only works if the archive is still available)
4. Returns `true` if **any single hash** matches

This means Terraform does **not** require all hashes to match — just one. Unrecognized hash schemes (future-proofing for `h2:`, etc.) are silently skipped rather than causing errors.

### Calculate h1 with Pure Bash (No Ruby)

Alternative to the ruby one-liner using `xxd` and `base64`:

```sh
cd .terraform/providers/registry.terraform.io/bpg/proxmox/0.69.0/linux_amd64

# Generate the sorted sha256sum list and hash it
sha256sum $(ls | sort) | sha256sum | awk '{print $1}' \
  | xxd -r -p | base64
# ULUO3AUJfhLxDU02ktVgVS24MH1XyvKkHAujs/KYI1I=
```

Or using Python:

```sh
sha256sum $(ls | sort) | sha256sum | awk '{print $1}' \
  | python3 -c "import sys,base64,binascii; print(base64.b64encode(binascii.unhexlify(sys.stdin.read().strip())).decode())"
```


## Where zh Hashes Come From

The `zh` entries can also be found in the provider's release within the `SHA256SUMS` file:

```sh
curl -sL https://github.com/bpg/terraform-provider-proxmox/releases/download/v0.69.0/terraform-provider-proxmox_0.69.0_SHA256SUMS
```

Each line in that file is the SHA256 hash of a platform-specific zip archive. Terraform uses these to verify the downloaded zip before extraction.


## Why zh Is Considered Legacy

The `zh:` scheme only works for `.zip` files downloaded through the main Terraform registry protocol. It cannot verify providers installed from:

- A local filesystem mirror
- A network mirror (non-registry)
- Providers built from source

The `h1:` scheme works regardless of how the provider was obtained because it hashes the unpacked directory contents, not the delivery format.


## Security Model

The hashes in the lock file protect against tampered downloads after the initial lock. However, the trust model relies on the first `terraform init` that wrote the lock file being performed in a trusted environment. Once hashes are recorded and committed to version control, subsequent runs verify integrity against those known-good values.

This means:
- The first `terraform init` is a trust-on-first-use (TOFU) operation
- After that, any alteration to the provider binary will be caught
- Signing verification (via the registry's GPG signatures) happens at download time, but the lock file itself stores only hashes, not signatures

### Lock File Tampering Risk

The lock file can contain **multiple** `h1:` and `zh:` hashes per provider (one per platform). Critically, the two hash types don't need to have any relationship to each other — Terraform only requires that the downloaded artifact matches *at least one* hash of either type.

This means an attacker who modifies a provider binary on disk could compute a new `h1:` hash for the tampered directory and add it alongside the legitimate hashes in the lock file. The existing hashes remain valid for other platforms, so the modification won't break anyone else's installation — it just silently allows the tampered binary.

**Mitigations:**

- Always commit `.terraform.lock.hcl` to version control
- Carefully review any diffs to the lock file — unexpected hash additions are a red flag
- Treat lock file changes in PRs with the same scrutiny as dependency upgrades
- Run `terraform init` only in trusted environments for the initial lock generation


## Multi-Platform Hashes

By default, Terraform only records hashes for your current platform. This causes problems in teams where developers use macOS locally but CI runs on Linux.

Add hashes for multiple platforms:

```sh
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64
```

This downloads the provider zip for each platform, computes both `zh:` and `h1:` hashes, and writes them all to `.terraform.lock.hcl`. Commit the result so all team members and CI can verify their platform-specific download.


## Providers Mirror (Air-Gapped Environments)

For environments without internet access, use `terraform providers mirror` to pre-download providers:

```sh
# Download all required providers to a local directory
terraform providers mirror /path/to/mirror

# Configure Terraform to use the mirror
# In ~/.terraformrc or terraform.rc:
provider_installation {
  filesystem_mirror {
    path = "/path/to/mirror"
  }
}
```

The lock file works the same way with mirrors — hashes are still verified against the recorded values.


## Troubleshooting

### Hash Mismatch Error

When Terraform detects a mismatch, you'll see an error like:

```
Error: Failed to install provider

Error while installing hashicorp/aws v5.46.0: the current package for
registry.terraform.io/hashicorp/aws 5.46.0 doesn't match any of the
checksums previously recorded in the dependency lock file.
```

Common causes:
- The lock file was generated on a different platform and lacks hashes for the current one
- The provider binary was corrupted during download
- A network proxy or mirror is serving a different file

### Lock File Has Only h1 but CI Needs zh (or Vice Versa)

This happens when the lock file was generated locally (producing `h1:` hashes) but CI installs from the registry (which checks `zh:` hashes). Fix it by regenerating with platform-specific hashes:

```sh
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64
```

### Regenerating a Corrupted Lock File

If the lock file is beyond repair, delete it and re-initialize:

```sh
rm .terraform.lock.hcl
terraform init
```

Then add multi-platform hashes if needed:

```sh
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64
```

Review and commit the new lock file.


## Update Terraform Providers

```sh
terraform init -upgrade
```

- This will ignore the `.terraform.lock.hcl` file.
- For plugins and modules, Terraform uses the newest installed version that meets the applicable constraints.
- When Terraform does not have an acceptable version of a required plugin or module (the `required_version` setting in the terraform block is missing), it attempts to download the newest version.

> You should never directly modify the lock file.


## References

- [Terraform Dependency Lock File](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
- [Lock and Upgrade Provider Versions](https://developer.hashicorp.com/terraform/tutorials/configuration-language/provider-versioning)
- [Version Constraints](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)
