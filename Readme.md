# Cert generate

```bash
# Install toolchain
curl -s -L -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -s -L -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod a+x /usr/local/bin/{cfssl,cfssljson}

pushd cert
# CA generate
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

# Server cert
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd server-csr.json | cfssljson -bare server

# Client cert
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd client-csr.json | cfssljson -bare client

chmod 777 *

popd
```

# Cluster status check
```bash
# MTLS check member status
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    -w table \
    member list

# TLS check member status
etcdctl --cacert=/opt/etcd/ssl/ca.pem \
    -w table \
    member list

# HTTP + MTLS put request check
curl -v --cacert "${ETCD_TRUSTED_CA_FILE}" \
    --cert "${ETCD_CLIENT_CERT_FILE}" --key "${ETCD_CLIENT_KEY_FILE}" \
    https://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello world"
```

# Test
```bash
# http + mtls test
curl -v --cacert ./cert/ca.pem \
    --cert ./cert/client.pem --key ./cert/client-key.pem \
    https://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello world"

# http + tls test
curl -v --cacert ./cert/ca.pem \
    https://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello world"
```


## enable auth
```bash
# export ETCD_TRUSTED_CA_FILE when mtls is disabled
export ETCD_TRUSTED_CA_FILE=/opt/etcd/ssl/ca.pem

# add role
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    role add rbac

# check role list
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    role list
    
# grant permission for role
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    role grant-permission rbac --prefix=true readwrite /pub

# grant permission for role
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    role get rbac
    

# add user
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    user add etcd.client --new-user-password="123456"
    
# check user list
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    user list

# grant permission
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    user grant-role etcd.client rbac

# check user roles
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    user get etcd.client
```

## enable auth
```bash
# add default root user
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    user add root --new-user-password="123456"

# enable auth
etcdctl --cacert="${ETCD_TRUSTED_CA_FILE}" \
    --cert="${ETCD_CLIENT_CERT_FILE}" --key="${ETCD_CLIENT_KEY_FILE}" \
    auth enable
```



## HTTP with tls test
```bash
# fetch etcd op token
curl -v --cacert ./cert/ca.pem \
    --cert ./cert/client.pem --key ./cert/client-key.pem \
    -L https://127.0.0.1:2379/v3/auth/authenticate \
    -X POST -d '{"name": "etcd.client", "password": "123456"}'
```
