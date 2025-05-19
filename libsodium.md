# Diagnosing and Fixing `crypto_vrf_ietfdraft13_*` Linking Errors in `cardano-node`

## ❗ Problem Summary

When building `cardano-node` 10.3.1, you may encounter linker errors such as:

```
undefined reference to `crypto_vrf_ietfdraft13_publickeybytes`
undefined reference to `crypto_vrf_ietfdraft13_secretkeybytes`
```

These errors indicate that the installed `libsodium` **does not support the VRF ietfdraft13 extension functions**, which are **required** for recent `cardano-node` builds (especially those involving `cardano-crypto-praos`).

---

## 🧪 Diagnosis Steps

1. Check the current system `libsodium`:

```bash
nm -D /usr/local/lib/libsodium.so | grep crypto_vrf_ietfdraft13
```

If **no output appears**, then the current library lacks support for VRF ietfdraft13 functions.

2. You may also try:

```bash
strings /usr/local/lib/libsodium.so | grep ietfdraft13
```

Again, if this returns nothing, your build is missing VRF extensions.

---

## ✅ Correct Source: IntersectMBO's `libsodium` Fork

The required functions are **not available** in the default IOG or upstream `libsodium`. Instead, use:

```
https://github.com/IntersectMBO/libsodium
```

### Recommended Commit:

```
dbb48ccf89bd3b6a292b16276fd9cfd6dd6675b2
```

---

## 🔧 Installation Instructions

```bash
# Clean existing versions
sudo rm -rf /usr/local/lib/libsodium* /usr/local/include/sodium

# Clone the correct repo
cd ~/libdeps
git clone https://github.com/IntersectMBO/libsodium
cd libsodium
git checkout dbb48cc

# Replace broken/missing config files
curl -o build-aux/config.guess https://git.savannah.gnu.org/gitweb/?p=config.git\;a=blob_plain\;f=config.guess\;hb=HEAD
curl -o build-aux/config.sub https://git.savannah.gnu.org/gitweb/?p=config.git\;a=blob_plain\;f=config.sub\;hb=HEAD
chmod +x build-aux/config.*

# Build
./autogen.sh
./configure --prefix=/usr/local
make -j$(nproc)
sudo make install
sudo ldconfig
```

---

## 🔁 Verify Installation

```bash
nm -D /usr/local/lib/libsodium.so | grep crypto_vrf_ietfdraft13
```

You should now see multiple symbols like:

```
crypto_vrf_ietfdraft13_prove
crypto_vrf_ietfdraft13_verify
crypto_vrf_ietfdraft13_publickeybytes
```

---

## ✅ Final Check

Rebuild `cardano-node`:

```bash
cd ~/cardano-src/cardano-node
cabal build cardano-node
```

Then check:

```bash
cardano-node --version
```

Should return:

```
cardano-node 10.3.1 - linux-x86_64 - ghc-9.6
```

---

## 🔚 Conclusion

This fix is **crucial** for anyone building the latest `cardano-node` and encountering undefined VRF symbols. Using the **IntersectMBO fork of libsodium** with the right commit ensures compatibility.

> This note was built during a real-world debugging session in May 2025. Please check Intersect/Cardano repos for updates if builds change.
