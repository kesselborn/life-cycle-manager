(c) 2018-2019 HomeAccessoryKid

### Instructions for end users:
TBD

### Instructions if you own the private key:
```
cd life-cycle-manager
```
- initial steps to be expanded

#### These are the steps if not introducing a new key pair
- for the example, we expect the current version to be `1.0.0` and the new version to be `1.0.1`
- create/update the file `versions1/latest-pre-release` without new-line and setup the new versions folder
```
# adjust these env vars
export CURRENT_VERSION=1.0.0 
export NEW_VERSION=1.0.1
export PORT=/dev/cu.SLAB_USBtoUART

git rm -rf versions1/${CURRENT_VERSION}v
mkdir versions1/${NEW_VERSION}v
echo -n ${NEW_VERSION} > versions1/${NEW_VERSION}v/latest-pre-release
cp versions1/certs.sector* versions1/${NEW_VERSION}v
cp versions1/public*key*   versions1/${NEW_VERSION}v
```
- set `local.mk` to the **ota-main** program
```
make -j6 rebuild OTAVERSION=${NEW_VERSION}
mv firmware/otamain.bin versions1/${NEW_VERSION}v
```
- set `local.mk` back to **ota-boot** program
```
make -j6 rebuild OTAVERSION=${NEW_VERSION}
mv firmware/otaboot.bin versions1/${NEW_VERSION}v
make -j6 rebuild OTAVERSION=${NEW_VERSION} OTABETA=1
cp firmware/otaboot.bin versions1/${NEW_VERSION}v/otabootbeta.bin
```
- commit the new version : `git add versions1/${NEW_VERSION}v && git commit -av && git push`
- release the version: `./release ${NEW_VERSION}v versions1/${NEW_VERSION}v`
- follow the link the script outputs at the end and set the release to `pre-release`

- erase the flash and upload the privatekey
```
esptool.py -p /dev/cu.usbserial-* --baud 230400 erase_flash 
esptool.py -p /dev/cu.usbserial-* --baud 230400 write_flash 0xf9000 versions1-privatekey.der
```
- upload the ota-boot BETA program to the device that contains the private key
```
make flash OTAVERSION=${NEW_VERSION} OTABETA=1
```
- power cycle to prevent the bug for software reset after flash  
- setup wifi and select the ota-demo repo without pre-release checkbox  
- create the 2 signature files next to the bin file and upload to github one by one  
- verify the hashes on the computer  
```
for f in versions1/${NEW_VERSION}v/*.bin
do
  openssl sha384 -binary -out ${f}.sig $f
  printf "%08x" `cat $f | wc -c`| xxd -r -p >> ${f}.sig
done
```

- upload the file versions1/1.0.0v/latest-pre-release to the 'latest release' assets on github

#### Testing

- test the release with several devices that have the beta flag set  
- if bugs are found, leave this release at pre-release and start a new version
#
- if the results are 100% stable  
- make the release a production release on github  
- remove the private key  
```
esptool.py -p /dev/cu.usbserial-* --baud 230400 write_flash 0xf9000 versions1/blank.bin
```

### How to make a new signing key pair

- for this description, we expect the current version to be `1.0.0v`, i.e. the repository contains the filder `version1/1.0.0.v`

- when creating a new private key, move the `version1` folder to some temp dir (you won't need it anymore) and create a new `version1` folder with the content of the old `versions1/1.0.0v` folder:

  ```
  mv versions1 /tmp
  mv /tmp/versions1/1.0.0v versions1
  mv 
  ```

- **if** `version1` contains files called something like `public-N.key.sig` and `public-N.key`, rename the files by increment `N` by one (i.e. `public-1.key` -> `public-2.key`, etc.)  

- execute `mv versions1/certs.sector versions1/public-2.key`

- note the new `certs.sector` is `public-1.key` but we never need to call it with that name  

- remove all the duplicates that will not change from the previous versions folder like `blank.bin` ...  

- note a `public-N.key.sig` is a signature on a `certs.sector` file, but using an older key  

- and `certs.sector.sig` is also a signature on a `certs.sector` file, but using its own key  

- there is no need to upload or even keep `public-N.key` for `versionsN` since it is never needed  

- make a new key pair
```
mkdir /tmp/ecdsa
cd    /tmp/ecdsa
openssl ecparam -genkey -name secp384r1 -out secp384r1prv.pem
openssl ec -in secp384r1prv.pem -outform DER -out secp384r1prv.der
openssl ec -in secp384r1prv.pem -outform DER -out secp384r1pub.der -pubout
xxd -p secp384r1pub.der > certs.hex
```
- save the `secp384r1prv.pem` in a secret place and destroy `secp384r1prv.pem` and `secp384r1prv.der` from `/tmp/ecdsa`

- back in the repository, create the new `certs.sector`
```
xxd -p -r /tmp/ecdsa/certs.hex > versions1/certs.sector
```
- start a new release as described above, but in the first run, use the previous private key in `0xf9000`
```
esptool.py -p /dev/cu.usbserial-* --baud 230400 write_flash 0xf9000 versionsN-1-privatekey.der
```
- collect public-1.key.sig and store it in the new version folder and copy it to versions1
```
cp  versions1/1.0.0v/public-1.key.sig versions1
```
- then flash the new private key
```
esptool.py -p /dev/cu.usbserial-* --baud 230400 write_flash 0xf9000 versions1-privatekey.der
```
- collect cert.sector.sig and store it in the new version folder and copy it to versions1 
```
cp  versions1/1.0.0v/certs.sector.sig versions1
```
- continue with a normal deployment to create the 2 signature files next to the bin files
