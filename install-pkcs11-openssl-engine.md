```bash
sudo apt-get -y install opensc libengine-pkcs11-openssl

cd
cat > pkcs11-engine.conf <<EOF
openssl_conf = openssl_def

[openssl_def]
engines = engine_section

[engine_section]
pkcs11 = pkcs11_section

[pkcs11_section]
engine_id = pkcs11

# This is the path to engine that openssl loads
dynamic_path = /usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so

# this is the path to the pkcs11 implementation that the pkcs11 engine loads
MODULE_PATH = /usr/local/lib/libtpm2_pkcs11.so
PIN=1234
#init = 0
EOF

# copy to root, as opensll will be executed with sudo
sudo cp $HOME/pkcs11-engine.conf /root
```