# PKI & TLS Setup Guide

## Overview

**Network Topology & Credentials**

| Role            | IP Address   | Username      | Password     |
| --------------- | ------------ | ------------- | ------------ |
| Trustpoint (CA) | 192.168.5.10 | `trustpoint`  | `testing321` |
| HTTP Server     | 192.168.5.20 | `httpserver`  | `testing321` |
| HTTP Client #1  | 192.168.5.30 | `httpclient1` | `testing321` |
| HTTP Client #2  | 192.168.5.31 | `httpclient2` | `testing321` |

---

# Section 1 – Server & Client Setup

## Client Setup

1. **Generate client key & certificate**

   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout httpclient<N>.key -x509 -days 365 -out httpclient<N>.pem -subj "/CN=httpclient<N>" -extensions v3_req -config client_ext.cnf
   ```
2. **Verify extension file**

   * Inspect `client_ext.cnf` for correct X.509 v3 extensions.
3. **Create PKCS#12 file**

   ```bash
   openssl pkcs12 -export -inkey httpclient<N>.key -in httpclient<N>.pem -out httpclient.p12 -name "httpclient<N>"
   ```

   * When prompted, use password: `testing321`
4. **Import into Firefox**

   * Settings → Search "cert" → View Certificates → Your Certificates
   * Import `httpclient.p12` and enter password `testing321`

## Server Setup

1. **Generate CA key & certificate**

   ```bash
   openssl genrsa -out myCA.key 2048
   openssl req -x509 -new -nodes -key myCA.key -sha256 -days 365 -subj "/CN=My Local CA" -out myCA.pem
   ```
2. **Generate server key & CSR**

   ```bash
   openssl genrsa -out priv_key.pem 2048
   openssl req -new -key priv_key.pem -subj "/CN=localhost" -out httpserver.csr
   ```
3. **Verify extension file**

   * Inspect `v3.ext` for required X.509 extensions.
4. **Create certificate from CSR**

   ```bash
   openssl x509 -req -in httpserver.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out httpserver.pem -days 825 -sha256 -extfile v3.ext
   ```
5. **Reload Apache**

   ```bash
   sudo systemctl reload apache2.service
   ```

---

# Section 2 – Certificate Distribution

## HTTP Server

1. Distribute CA certificate to clients

   ```bash
   scp myCA.pem httpclient1@192.168.5.30:~
   scp myCA.pem httpclient2@192.168.5.31:~
   ```

## Client

1. Distribute client certificates to server

   ```bash
   scp httpclient1.pem httpserver@192.168.5.20:~
   scp httpclient2.pem httpserver@192.168.5.20:~
   ```

---

# Section 3 – Final Configuration

## Client (Firefox)

1. Settings → Search "cert" → View Certificates → Authorities
2. Import `myCA.pem` → enable all trust options → OK

## Server (Apache)

1. **Create truststore**

   ```bash
   cat httpclient1.pem httpclient2.pem > client-trust-store.pem
   sudo chmod 664 client-trust-store.pem
   ```
2. **Restart Apache**

   ```bash
   sudo systemctl restart apache2.service
   ```

---

# Section 4 – Connection Verification

## Client
   1. Open browser at `https://192.168.5.20`
   2. Accept client certificate → OK
   3. Confirm **"Secure Connection Established"** appears

---

# Optional: Certificate Renewal

* Assume your client or TLS Server certificate is about to expire.
If time available. Try to replace the certs you created

---

# Final Section – PKI & CMP Enrollment

* **Trustpoint UI:** [https://192.168.5.10](https://192.168.5.10) (admin / testing321)

## Client

1. Create new device:

   * **Device:** HttpClient<N> (replace <N> with 1 or 2)
   * **Serial Number:** your choice
   * **Domain:** Arburg
   * **Select:** No domain credential
   * **Enrollment Method:** CMP Shared Secret
   * **Create Device**

2. Follow trustpoint instructions
   * **WARNING** Replace IP-Adress
3. Create PKCS#12 file:

   ```bash
   openssl pkcs12 -export -inkey key.pem -in cert.pem -certfile chain_without_root.pem -name "httpclient<N>" -out httpclient<N>.p12
   ```

   * Use password: `testing321`
4. Import into Firefox:

   * Import `full_chain.pem` under Authorities
   * Import client `.p12` under Your Certificates

## Server

* **Trustpoint UI:** [https://192.168.5.10](https://192.168.5.10) (admin / testing321)

1. Create new device:

   * **Name:** HttpServer
   * **Serialnumber:** your choice
   * **Domain:** Arburg
   * **Select:** No domain credential
   * **Enrollment:** CMP Shared Secret
   * **Create Device**
2. Follow help instructions
   * **WARINING: Replace IP-Adress with correct one**
   * **Add:** `192.168.5.20` to SAN

3. Rename output files:
   ```bash
   mv full_chain.pem client-trust-store.pem
   mv cert.pem httpserver.pem
   mv key.pem priv_key.pem
   ```
3. Restart Apache:

   ```bash
   sudo systemctl restart apache2.service
   ```
