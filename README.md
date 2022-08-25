# iotedge-edgeca-tpm
Some notes to guide you through the creation of a sample edgeCA certificate signed by a private CA, with the edgeCA private key being stored in a TPM via PKCS#11.

This applies to Azure IoT Edge 1.2+

## pre-requisites
* a linux machine
* Ubuntu 20.04
* IoT Edge 1.2+ (install instructions [here](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-provision-single-device-linux-symmetric?view=iotedge-2020-11&tabs=azure-portal%2Cubuntu#install-iot-edge))
* TPM/PKCS#11 stack with a HW or simulated TPM (see [here](install-tpm-stack.md))
* PKCS#11 openssl engine (see [here](install-pkcs11-openssl-engine.md))
  
## steps

### 1) remove any pre-existing certs
```bash
# folder with the certGen.sh script
rm -rf $HOME/certgen-home

# iotedge certs
sudo rm -rf /var/lib/aziot/certd/certs
```

### 2) create private CA (root and intermediate)
The IoT Edge's certGen.sh script is used.

```bash
cd
mkdir certgen-home
cd certgen-home

# get config
curl https://raw.githubusercontent.com/Azure/iotedge/main/tools/CACertificates/openssl_root_ca.cnf -o openssl_root_ca.cnf

# get script
curl https://raw.githubusercontent.com/Azure/iotedge/main/tools/CACertificates/certGen.sh -o certGen.sh
chmod +x certGen.sh

# create root
./certGen.sh create_root_and_intermediate

# copy the bundle (root+int) to /etc/aziot
sudo cp certs/azure-iot-test-only.intermediate-full-chain.cert.pem /etc/aziot/trust-bundle.pem
```

### 3) create the keypair in TPM/PKCS#11
```bash
export PKCS11_LIB_PATH='/usr/local/lib/libtpm2_pkcs11.so'

# create keypair with name "myrsa" and PIN "1234" in the TPM
sudo pkcs11-tool --module $PKCS11_LIB_PATH --label="myrsa" --login --pin 1234 --keypairgen --usage-sign

# sanity check: list objects
sudo tpm2_ptool listobjects --label edge --path '/opt/tpm2-pkcs11'
sudo tpm2_ptool listtokens --pid 1 --path '/opt/tpm2-pkcs11'
```

### 4) create edge-ca CSR and get it signed
Edit "EDGE_CA_CN" and "USER_HOME" to match yours.

```bash
sudo su

# do edit to match your needs
EDGE_CA_CN=myEdgeCAtpm-2.ca
USER_HOME=/home/arlotito

# 
PKCS11_ENGINE_CONF=${USER_HOME}/pkcs11-engine.conf
CERTS_HOME=${USER_HOME}/certgen-home

# create CSR:
export CERTIFICATE_OUTPUT_DIR=$(pwd)
OPENSSL_CONF=$PKCS11_ENGINE_CONF openssl req -new -sha256 \
	-engine pkcs11 \
	-keyform engine \
	-key "pkcs11:token=edge;object=myrsa;type=private" \
	-config $CERTS_HOME/openssl_root_ca.cnf \
	-subj /CN=$EDGE_CA_CN \
	-out $CERTS_HOME/csr/${EDGE_CA_CN}.csr.pem

SIGNING_CA_CERT=$CERTS_HOME/certs/azure-iot-test-only.intermediate.cert.pem
SIGNING_CA_PK=$CERTS_HOME/private/azure-iot-test-only.intermediate.key.pem

# sign certificate
openssl ca \
-batch \
-config $CERTS_HOME/openssl_root_ca.cnf \
-extensions v3_intermediate_ca \
-days 30 \
-notext \
-md sha256 \
-cert $SIGNING_CA_CERT \
-keyfile $SIGNING_CA_PK \
-passin pass:1234 \
-keyform PEM \
-in $CERTS_HOME/csr/${EDGE_CA_CN}.csr.pem \
-out $CERTS_HOME/certs/${EDGE_CA_CN}.cert.pem \
-outdir $CERTS_HOME/newcerts
```

### 5) copy edge-ca cert to /etc/aziot
```bash
# edgeca cert
sudo cp $CERTS_HOME/certs/${EDGE_CA_CN}.cert.pem /etc/aziot/edge-ca.cert.pem
```

### 6) optionally verify edge-ca vs trust bundle
```bash
cd /etc/aziot
openssl verify -show_chain -CAfile trust-bundle.pem edge-ca.cert.pem
```

and the result should be:
```bash
edge-ca.cert.pem: OK
Chain:
depth=0: CN = myEdgeCAtpm.ca (untrusted)
depth=1: CN = Azure_IoT_Hub_Intermediate_Cert_Test_Only
depth=2: CN = Azure_IoT_Hub_CA_Cert_Test_Only
```

### 7) edit config.toml
First of all, let's configure the PKCS#11:
```bash
[aziot_keys]
pkcs11_lib_path = "/usr/local/lib/libtpm2_pkcs11.so"
pkcs11_base_slot = "pkcs11:token=edge?pin-value=1234"
```

The edge-ca certificate is on the filesystem, while the corresponding private key is stored in the TPM/PKCS#11
```bash
[edge_ca]
cert = "file:///etc/aziot/edge-ca.cert.pem"
pk = "pkcs11:token=edge;object=myrsa?pin-value=1234"
```

The "edge-ca.cert.pem" includes the the individual edge-ca certificate WITHOUT the issuing chain (root/intermediate).

The issuing bundle (i.e. root/intermediate) is stored in file "/etc/aziot/trust-bundle.pem" and must be set as follows:

```bash
trust_bundle_cert = "file:///etc/aziot/trust-bundle.pem"
```

### 8) apply the configuration
```bash
sudo iotedge config apply
```

### 9) check the logs
```bash
iotedge logs edgeHub
```

and you should see something like:
```bash
2022-08-12 09:08:05  Starting Edge Hub
2022-08-12 09:08:05.355 +00:00 Edge Hub Main()
<6> 2022-08-12 09:08:05.736 +00:00 [INF] - Installing certificates [CN=myEdgeCAtpm-2.ca:09/11/2022 08:58:35] to Root
<6> 2022-08-12 09:08:05.771 +00:00 [INF] - Installing certificates [CN=myEdgeCAtpm-2.ca:09/11/2022 08:58:35],[CN=Azure_IoT_Hub_Intermediate_Cert_Test_Only:09/11/2022 08:46:01],[CN=Azure_IoT_Hub_CA_Cert_Test_Only:09/11/2022 08:46:01] to Root
```

...confirming that the edge-ca certificate is installed along with the issuing chain (trust-bundle)

## Troubleshooting

### TLS AUTH error on a module
If you deploy a module (e.g. the Simulated Temperature module) and you see an error like this in its log:
```bash
[Information]: Trying to initialize module client using transport type [Amqp_Tcp_Only].
Unhandled exception. System.AggregateException: One or more errors occurred. (TLS authentication error.)
 ---> System.Security.Authentication.AuthenticationException: TLS authentication error.
```
it means that something is wrong with the certificate chain and/or config.toml. 

Please go back to [step 6](#6-optionally-verify-edge-ca-vs-trust-bundle) to check the chain, and [step 7](#7-edit-configtoml) to configure the config.toml

### Connection refused /var/run/iotedge/workload.sock
If one of your edge modules fails with the following error:
```bash
Unhandled exception. System.AggregateException: One or more errors occurred. (Connection refused /var/run/iotedge/workload.sock)
```
re-deploy the module.
