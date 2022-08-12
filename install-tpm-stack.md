# How to install

Grab the scripts from https://github.com/arlotito/iotedge-tpm2cloud:
```
git clone https://github.com/arlotito/iotedge-tpm2cloud
cd iotedge-tpm2cloud/scripts-provisioning
```

Install the TPM/PKCS#11 stack including a simulated SW TPM
```
./tpm2-stack-install.sh ubuntu2004_amd64 swtpm
```
NOTE: in case you have an HW tpm, replace "swtpm" with "hwtpm"

(re)init the PKCS11 store
```
sudo su

TOKEN='edge'
PIN='1234'

tpm2_clear

rm -rf /opt/tpm2-pkcs11
mkdir -p /opt/tpm2-pkcs11

tpm2_ptool init --primary-auth '1234' --path /opt/tpm2-pkcs11
tpm2_ptool addtoken --path /opt/tpm2-pkcs11 \
--sopin "so$PIN" --userpin "$PIN" \
--label "$TOKEN" --pid '1'
```

fix permissions to get access to the PKCS11 store:
```
sudo chown "aziotks:aziotks" /opt/tpm2-pkcs11 -R
sudo chmod 0700 /opt/tpm2-pkcs11
```

Optionally list the tokens:
```
sudo tpm2_ptool listtokens --pid 1 --path '/opt/tpm2-pkcs11'
```