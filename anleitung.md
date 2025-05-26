# Allgemein
- Netzwerk:
    - Trusptoint:   192.168.5.10
        - User: trustpoint
        - pw:   testing321
    - HTTP-Server:  192.168.5.20
        - User: httpserver
        - pw:   testing321
    - HTTP-Client1: 192.168.5.30
        - User: httpclient1
        - PW:   testing321
    - HTTP-Client2: 192.168.5.31
        - User: httpclient2
        - PW:   testing321

# Section 1 - Server / Client Einrichtung

## Einrichtung - Server

1. Server Zertifikat erstellen
    - **Root RSA key:** `openssl genrsa -out myCA.key 2048`

    - **Root Cert:** `openssl req -x509 -new -nodes -key myCA.key -sha256 -days 3650 -subj "/CN=My Local CA" -out myCA.pem`

    - **RSA Key for self sigend cert:** `openssl genrsa -out priv_key.pem 2048`

    - **CSR for self signed TLS Zertifikat:** `openssl req -new -key priv_key.pem -subj "/CN=localhost" -out httpserver.csr `

2. Extensions (v3.ext) in file v3.ext überprüfen

3. Zertifikat aus CSR erstellen
    -`openssl x509 -req -in httpserver.csr -CA myCA.pem -CAkey myCA.key -CAcreateserial -out httpserver.pem -days 825 -sha256 -extfile v3.ext`

4. Apache neu starten
    - `sudo systemctl reload apache2.service`

## Einrichtung - Client

1. Client Key + Zertifikate erstellen
    - **Generate Cert** (replace N with 1 or 2 depending on client)**:** `openssl req -newkey rsa:2048 -nodes -keyout httpclientN.key -x509 -days 365 -out httpclientN -subj "/CN=httpclientN" -extensions v3_req -config client_ext.cnf`

2. Extension (client_ext.cnf) überprüfen

3. Create pkcs12 file
    - **Create pkcs12** (replace N with 1 or 2)**:** `openssl pkcs12 -export -inkey httpclientN.key -in httpclientN.pem httpclient.p12 -name "httpclientN"`
    - If asked for password, Use password testing321

4. Import in browser (Firefox)
    - settings
    - search for "cert"
    - click on view certificates
    - click on "Your Certificates"
    - Import pkcs12 FIle (httpclient.p12)
    - If asked for password, Use password testing321