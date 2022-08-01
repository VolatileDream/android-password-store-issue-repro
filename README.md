# Pass <-> Android Password Store incompatibility

Steps to highlight a compatibility issue between [pass] and [Android-Password-Store].


## Steps

1. Create GPG Key (`gpg-key-9FFAABCC1F7D00CE40A1EE1FED31C0DBA11F5155.asc`)
2. Add two (2) encrypting subkeys, so the GPG key has a total of three encryption capable keys:
    - `elg3072/7F51EC6D028FE1FD` (new)
    - `rsa3072/6E7B4FAC4385056B` (new)
    - `rsa3072/50D676A945F7873F` (from creation)
3. Setup and create [pass] password store
    - `> export PASSWORD_STORE_DIR=$PWD/password-store`
    - `> pass init 9FFAABCC1F7D00CE40A1EE1FED31C0DBA11F5155`
    - (At this point [pass] sees the git repo and starts creating commits too)
4. Add an entry to the [pass] store
    - `> echo "abc 123" | pass insert -m test-BA11F5155`
5. Ask GPG about the stored password file `password-store/test-BA11F5155.gpg`
    - `> gpg --list-packets password-store/test-BA11F5155.gpg`
      ```
      gpg: encrypted with 3072-bit RSA key, ID 6E7B4FAC4385056B, created 2022-08-01
            "Android Password Store Reproduction <VolatileDream@users.noreply.github.com>"
      # off=0 ctb=85 tag=1 hlen=3 plen=396
      :pubkey enc packet: version 3, algo 1, keyid 6E7B4FAC4385056B
        data: [3070 bits]
      # off=399 ctb=d2 tag=18 hlen=2 plen=57 new-ctb
      :encrypted data packet:
        length: 57
        mdc_method: 2
      # off=420 ctb=cb tag=11 hlen=2 plen=14 new-ctb
      :literal data packet:
        mode b (62), created 1659372432, name="",
        raw data: 8 bytes
      ```
    - Notice the file was encrypted to one (1) key by GPG: `6E7B4FAC4385056B`
6. Attempt to make [pass] encrypt to all the encryption subkeys for `A11F5155`
    - `> echo -e '50D676A945F7873F\n6E7B4FAC4385056B\n7F51EC6D028FE1FD' > password-store/.gpg-id`
    - `> echo test-multi-key | pass insert -m test-three-key`
    - `> gpg --list-packets password-store/test-three-key.gpg`
    ```
      gpg: encrypted with 3072-bit RSA key, ID 6E7B4FAC4385056B, created 2022-08-01
            "Android Password Store Reproduction <VolatileDream@users.noreply.github.com>"
      # off=0 ctb=85 tag=1 hlen=3 plen=396
      :pubkey enc packet: version 3, algo 1, keyid 6E7B4FAC4385056B
        data: [3072 bits]
      # off=399 ctb=d2 tag=18 hlen=2 plen=64 new-ctb
      :encrypted data packet:
        length: 64
        mdc_method: 2
      # off=420 ctb=cb tag=11 hlen=2 plen=21 new-ctb
      :literal data packet:
        mode b (62), created 1659373364, name="",
        raw data: 15 bytes
    ```
    - Notice that this has **not** worked, and only one key was used: `6E7B4FAC4385056B`
7. Investigate GPG behaviour when specifying multiple subkeys
    - `> gpg -r 50D676A945F7873F -r 6E7B4FAC4385056B -r 7F51EC6D028FE1FD --encrypt README.md`
    ```
      gpg: 6E7B4FAC4385056B: skipped: public key already present
      gpg: 50D676A945F7873F: skipped: public key already present
    ```
    - `> gpg --list-packets README.md.gpg`
    ```
      gpg: encrypted with 3072-bit RSA key, ID 6E7B4FAC4385056B, created 2022-08-01
            "Android Password Store Reproduction <VolatileDream@users.noreply.github.com>"
      # off=0 ctb=85 tag=1 hlen=3 plen=396
      :pubkey enc packet: version 3, algo 1, keyid 6E7B4FAC4385056B
        data: [3072 bits]
      # off=399 ctb=d2 tag=18 hlen=2 plen=0 partial new-ctb
      :encrypted data packet:
        length: unknown
        mdc_method: 2
      # off=420 ctb=a3 tag=8 hlen=1 plen=0 indeterminate
      :compressed packet: algo=2
      # off=422 ctb=ad tag=11 hlen=3 plen=2845
      :literal data packet:
        mode b (62), created 1659373837, name="README.md",
        raw data: 2830 bytes
    ```
    - Notice, still only one (1) encryption key was used: `6E7B4FAC4385056B`
    - `> rm README.md.gpg` (cleanup)

[pass]: https://www.passwordstore.org/
[Android-Password-Store]: https://github.com/android-password-store/Android-Password-Store/
