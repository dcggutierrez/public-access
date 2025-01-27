# Setting Up TLS with LDAP Authentication and PostgreSQL Encryption

## Overview
This guide provides step-by-step instructions to:

1. Enable TLS for internal connections to LDAP for staff account authentication.
2. Configure PostgreSQL to use TLS for encrypted communication.
3. Use Let's Encrypt with `cert-manager` for automating certificate management.
4. Alternatively, use self-signed certificates if automation is not feasible.

---

## Prerequisites

- Access to [certificate generation](https://git.ustapps.com/pki/x509) and [import tools](https://git.ustapps.com/pki/trustca).
- Scripts for generating certificates: `x509`.
- Scripts for importing certificates: `trustca`.
- Root or admin privileges to install and configure certificates.
- Kubernetes cluster with `cert-manager` installed (if using Let's Encrypt).

---

## Steps

### 1. Install `cert-manager`

To automate certificate management, install `cert-manager` in your Kubernetes cluster:
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/vX.Y.Z/cert-manager.yaml
```
Replace `vX.Y.Z` with the latest version of `cert-manager`.

---

### 2. Create Certificate Issuers

#### 2.1 Create a ClusterIssuer for Let's Encrypt

For production environments, define a `ClusterIssuer` resource:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

### 3. Request Certificates

#### 3.1 Generate Certificates for LDAP

Create a `Certificate` resource for LDAP:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ldap-tls
  namespace: default
spec:
  secretName: ldap-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: ldap.example.com
  dnsNames:
  - ldap.example.com
```

#### 3.2 Generate Certificates for PostgreSQL

Create a `Certificate` resource for PostgreSQL:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: postgres-tls
  namespace: default
spec:
  secretName: postgres-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: postgres.example.com
  dnsNames:
  - postgres.example.com
```

---

### 4. Configure Services to Use TLS

#### 4.1 Configure LDAP to Use TLS
Mount the `ldap-tls-secret` in the LDAP container and update the configuration to use the mounted certificate and private key:
- Certificate: `/etc/ldap/tls/tls.crt`
- Key: `/etc/ldap/tls/tls.key`
- CA Chain: `/etc/ldap/tls/ca.crt`

Example Kubernetes manifest snippet:
```yaml
volumes:
  - name: ldap-tls
    secret:
      secretName: ldap-tls-secret
volumeMounts:
  - mountPath: /etc/ldap/tls
    name: ldap-tls
    readOnly: true
```

#### 4.2 Configure PostgreSQL to Use TLS
Mount the `postgres-tls-secret` in the PostgreSQL container and update `postgresql.conf`:
```conf
ssl = on
ssl_cert_file = '/etc/postgres/tls/tls.crt'
ssl_key_file = '/etc/postgres/tls/tls.key'
ssl_ca_file = '/etc/postgres/tls/ca.crt'
```

Example Kubernetes manifest snippet:
```yaml
volumes:
  - name: postgres-tls
    secret:
      secretName: postgres-tls-secret
volumeMounts:
  - mountPath: /etc/postgres/tls
    name: postgres-tls
    readOnly: true
```

#### 4.3 Configure Applications Connecting to PostgreSQL
Ensure application containers have the CA certificate installed and configure connection URLs to use `sslmode=verify-ca`.

---

### 5. Verify Configuration

1. **Test LDAP TLS Connection:**
   ```bash
   ldapsearch -H ldaps://ldap.example.com -x -b "dc=example,dc=com"
   ```

2. **Test PostgreSQL TLS Connection:**
   ```bash
   psql "sslmode=verify-ca host=postgres.example.com user=<username> dbname=<dbname>"
   ```

---

### 6. Use Self-Signed Certificates (Optional)

If Let's Encrypt is not an option, use the `x509` script to generate self-signed certificates. Follow the steps below:

#### 6.1 Create a Root Certificate Authority (CA)
```bash
./newcert.sh -t ca root
```

#### 6.2 Generate and Sign Certificates
**For LDAP:**
```bash
./newcert.sh -ca root ldap.example.com
```
**For PostgreSQL:**
```bash
./newcert.sh -ca root postgres.example.com
```

#### 6.3 Import Certificates
Use the `trustca` script to import the self-signed CA certificate into trusted stores:
```bash
CERTIMPORT_SYSTEM="debian" certimport.sh /path/to/ca/certs
```

#### 6.4 Distribute Certificates to Containers
Manually mount the CA certificate in all containers communicating with LDAP or PostgreSQL.

---

### 7. Automate Certificate Renewal

- For Let's Encrypt, certificates are renewed automatically by `cert-manager`.
- For self-signed certificates, periodically regenerate and re-import certificates using the `x509` and `trustca` scripts.

---

### Notes
- Ensure all containers trust the CA certificate when using self-signed certificates.
- Test connections regularly to verify TLS encryption.
