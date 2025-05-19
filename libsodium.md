# Debug Notes: Building `libsodium` with VRF support for Cardano

## ✅ Summary

This document outlines the steps taken to properly build `libsodium` with VRF (Verifiable Random Function) support required by `cardano-crypto-praos` during Cardano node compilation (e.g., version 10.3.1 with GHC 9.6.7). It also records common issues and their fixes.

---

## ⚠️ Problem

### Error During Linking

```
(.text+0x97ba): undefined reference to `crypto_vrf_ietfdraft13_publickeybytes'
...
undefined reference to `crypto_vrf_ietfdraft13_secretkeybytes'
```

This means the currently installed `libsodium` does **not contain the required VRF symbols**.

### Cause

The default `libsodium` or the version from `input-output-hk/libsodium` does **not** include VRF support. VRF functionality (e.g., `crypto_vrf_ietfdraft13_*`) is only available in custom forks used by Intersect or Input Output.

---

## 🧱️ Solution: Use the Correct Source

### ✅ Correct Repository

Use the VRF-enabled fork from Intersect:

```
https://github.com/IntersectMBO/libsodium
```

We recommend checking out a specific working branch:

```bash
git clone https://github.com/IntersectMBO/libsodium
cd libsodium
git checkout iquerejeta/vrf_batchverify
```

---

## 🧱 Build Instructions (WSL / Linux)

```bash
./autogen.sh
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
sudo ldconfig
```

---

## ✅ Verification

After installation, confirm VRF functions exist:

```bash
nm -D /usr/local/lib/libsodium.so | grep ietfdraft13
```

You should see lines like:

```
crypto_vrf_ietfdraft13_batch_verify
crypto_vrf_ietfdraft13_secretkeybytes
...
```

---

## 🌐 Note on Savannah Download Failures

During this process, `autoreconf` or `autogen.sh` may attempt to download updated versions of `config.guess` and `config.sub` from:

```
https://git.savannah.gnu.org/cgit/config.git/plain/config.sub
```

### ❌ Problem:

As of 2025-05-19, this site returned a **502 Bad Gateway** response intermittently or continuously.

### ✅ Workaround:

Use the GitHub mirror instead:

```bash
curl -o build-aux/config.guess https://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD
curl -o build-aux/config.sub https://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub;hb=HEAD
chmod +x build-aux/config.*
```

Or use a system copy if available:

```bash
cp /usr/share/misc/config.guess build-aux/
cp /usr/share/misc/config.sub build-aux/
```

---

## ✅ Final Outcome

After using the correct fork and fixing `config.sub/config.guess`, the Cardano node compiled successfully with full VRF support:

```bash
cardano-node --version
# cardano-node 10.3.1 - linux-x86_64 - ghc-9.6
```
