# Diagnosing and Fixing Missing `crypto_vrf_ietfdraft13_*` Symbols When Building `cardano-node`

This guide is intended for developers who encounter linker errors related to missing `crypto_vrf_ietfdraft13_*` symbols during the build process of `cardano-node`. This issue often arises from linking against the **wrong version** of `libsodium`.

---

## ❗ Symptoms

When running `cabal build all` or similar commands to compile `cardano-node`, you may encounter errors like:

```bash
(.text+0x97ba): undefined reference to `crypto_vrf_ietfdraft13_publickeybytes'
(.text+0xa37a): undefined reference to `crypto_vrf_ietfdraft13_secretkeybytes'
collect2: error: ld returned 1 exit status
```

---

## ✅ Correct Diagnosis

These errors indicate that your current `libsodium` **does not include VRF draft13 support**. This feature is **only present** in certain forks of `libsodium`, such as the one maintained by [IntersectMBO](https://github.com/IntersectMBO/libsodium).

---

## 🧪 Quick Diagnostic Commands

```bash
nm -D /usr/local/lib/libsodium.so | grep ietfdraft13
# or
strings /usr/local/lib/libsodium.so | grep ietfdraft13
```

If **no output**, your `libsodium` is **missing VRF support**.

---

## 🛠️ Solution Steps

### 1. Remove old version
```bash
sudo rm -rf /usr/local/lib/libsodium* /usr/local/include/sodium
sudo ldconfig
```

### 2. Clone the correct `libsodium` fork
```bash
git clone https://github.com/IntersectMBO/libsodium.git
cd libsodium
git checkout iquerejeta/vrf_batchverify  # or the latest commit with VRF draft13
```

### 3. Regenerate and configure build system
```bash
./autogen.sh
curl -o build-aux/config.guess https://git.savannah.gnu.org/gitweb/?p=config.git\;a=blob_plain\;f=config.guess\;hb=HEAD
curl -o build-aux/config.sub https://git.savannah.gnu.org/gitweb/?p=config.git\;a=blob_plain\;f=config.sub\;hb=HEAD
chmod +x build-aux/config.*
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
sudo ldconfig
```

### 4. Verify symbols
```bash
nm -D /usr/local/lib/libsodium.so | grep ietfdraft13
```
Expected output includes:
```
crypto_vrf_ietfdraft13_publickeybytes
crypto_vrf_ietfdraft13_secretkeybytes
...
```

---

## 🔁 Rebuild Cardano Components
If `libsodium` was missing symbols before, rebuild dependencies:

```bash
cabal clean
cabal update
cabal build all
```

---

## 🙋 FAQ

### Q: Why doesn’t the official `input-output-hk/libsodium` have this?
A: It’s an outdated fork. Use `IntersectMBO/libsodium` for current development.

### Q: Why did `./configure` fail with 502 errors?
A: `config.sub` and `config.guess` are sometimes unreachable from GNU servers. Replace them manually via `curl` or `wget` as shown above.

---

## 🧭 Final Notes
- Always confirm symbol presence **before** building `cardano-node`
- Use `nm` and `strings` to verify correct linkage
- Track current forks and active branches in Cardano dev channels

---

## ✨ Credits
- Issue identified by [@Minhcardanian](https://github.com/Minhcardanian)
- Solution tested on Ubuntu 22.04 via WSL
