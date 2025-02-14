# TLS Setup Instructions for Postgres and OpenLDAP in a Kubernetes Datacenter

This document details how to secure TLS connections between Postgres and OpenLDAP using your X.509 certificate tools. It also explains which servers should run the certificate scripts.

---

## Servers & Where to Run the Scripts

- **Control Server (CA/Certificate Management Server):**  
  Run the **certificate signing tool** (`newcert.sh`) here. This server is your “CA host” where you generate the root CA, intermediate CA, and sign server certificates for Postgres and OpenLDAP.

- **Postgres & OpenLDAP Servers (or their corresponding Kubernetes Pods):**  
  Run the **certificate import tool** (`certimport.sh`) on these servers to install your CA (and other certs) into the system trust store.  
  - For Java-based apps (if any), run the import for JKS as needed.
  
- **Kubernetes Control (Optional):**  
  In Kubernetes, you typically generate certs on your CA/Control server, then create Secrets that are mounted into your Postgres and OpenLDAP pods. You may also have init containers running the import script if needed.

---

## Step-by-Step TLS Setup

### 1. Generate Certificates (Run on Control Server)

1. **Set Up Working Directory & Environment:**
   ```bash
   mkdir ~/certs && cd ~/certs
   export UST_X509_HOME=$(pwd)
   ```

2. **Create a Root CA:**
   ```bash
   ./newcert.sh -t ca root
   ```
   - This creates a root CA under `ca/root/`.

3. **Create an Intermediate CA:**
   ```bash
   ./newcert.sh -t ca -ca root intermediate
   ```
   - This creates an intermediate CA under `ca/intermediate/`.

4. **Generate a Server Certificate for Postgres:**
   ```bash
   ./newcert.sh -ca intermediate postgres.example.com
   ```
   - Answer prompts to add SAN or DNS entries as needed.
   - Certificate files will be stored in `server/postgres.example.com/`.

5. **Generate a Server Certificate for OpenLDAP:**
   ```bash
   ./newcert.sh -ca intermediate ldap.example.com
   ```
   - Certificate files will be stored in `server/ldap.example.com/`.

---

### 2. Import Certificates into Trusted Stores (Run on Target Servers)

1. **Copy Certificate Files:**  
   Gather the necessary files (for example, the CA certificate `ca.pem`, the server certs, and their private keys) into a directory on the target server.

2. **Import into System Trust Store (Debian Example):**
   ```bash
   CERTIMPORT_SYSTEM="debian" certimport.sh /path/to/certs/dir
   ```
   - Run this on your Postgres and OpenLDAP servers (or within a container’s startup script) to install your CA certificate into the trusted CA store.

3. **(Optional) Import into Java Keystore:**  
   If you need Java apps to trust your certs:
   ```bash
   CERTIMPORT_JKS="/opt/java/jdk/jre/lib/security/cacerts:changeit" certimport.sh /path/to/certs/dir
   ```

---

### 3. Deploy Certificates in Kubernetes

1. **Create Kubernetes Secrets:**

   - For **Postgres**:
     ```bash
     kubectl create secret generic postgres-tls \
       --from-file=postgres.crt=server/postgres.example.com/postgres.pem \
       --from-file=postgres.key=server/postgres.example.com/postgres.rsa \
       --from-file=ca.crt=ca/root/certs/ca.pem
     ```

   - For **OpenLDAP**:
     ```bash
     kubectl create secret generic ldap-tls \
       --from-file=ldap.crt=server/ldap.example.com/ldap.pem \
       --from-file=ldap.key=server/ldap.example.com/ldap.rsa \
       --from-file=ca.crt=ca/root/certs/ca.pem
     ```

2. **Mount Secrets in Your Pod Manifests:**  
   Update your Deployment/StatefulSet YAML files to mount these secrets as volumes. For example:
   ```yaml
   volumes:
     - name: tls-certs
       secret:
         secretName: postgres-tls
   ```

---

### 4. Configure Applications for TLS

1. **For Postgres:**

   - Edit your `postgresql.conf`:
     ```conf
     ssl = on
     ssl_cert_file = '/path/to/mounted/postgres.crt'
     ssl_key_file  = '/path/to/mounted/postgres.key'
     ssl_ca_file   = '/path/to/mounted/ca.crt'
     ```
   - Update `pg_hba.conf` to enforce SSL connections.

2. **For OpenLDAP:**

   - Update your LDAP configuration (e.g., in `slapd.conf` or using cn=config):
     ```conf
     TLSCertificateFile     /path/to/mounted/ldap.crt
     TLSCertificateKeyFile  /path/to/mounted/ldap.key
     TLSCACertificateFile   /path/to/mounted/ca.crt
     ```
   - Restart or reload the LDAP service to apply changes.

---

### 5. Test the TLS Setup

- **Test Postgres TLS Connection:**
  ```bash
  psql "host=postgres.example.com sslmode=require dbname=yourdb"
  ```

- **Test OpenLDAP TLS Connection:**
  ```bash
  ldapsearch -H ldaps://ldap.example.com -ZZ -b "dc=example,dc=com"
  ```

---

## Summary

- **Control Server:** Run `newcert.sh` to generate and sign certificates.
- **Postgres & OpenLDAP Servers:** Run `certimport.sh` (or use Kubernetes Secrets) to import certs into the trust store, then update your application configurations for TLS.
- **Kubernetes:** Create Secrets from the generated certs and mount them in your pods.

Follow these steps carefully and adjust paths, hostnames, and other settings to match your environment.

---

*If you have any questions or need further details, feel free to reach out!*