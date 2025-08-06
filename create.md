Berikut adalah **step-by-step perintah (command)** lengkap dari tutorial **Simple PKI** berbasis OpenSSL — bisa langsung kamu jalankan di terminal:

---

## 🔧 **0. Siapkan File dan Direktori**

```bash
git clone https://bitbucket.org/stefanholek/pki-example-1
cd pki-example-1
```

---

## 🏛️ **1. Membuat Root CA**

### 1.1 Buat direktori

```bash
mkdir -p ca/root-ca/private ca/root-ca/db crl certs
chmod 700 ca/root-ca/private
```

### 1.2 Buat file database

```bash
cp /dev/null ca/root-ca/db/root-ca.db
echo 01 > ca/root-ca/db/root-ca.crt.srl
echo 01 > ca/root-ca/db/root-ca.crl.srl
```

### 1.3 Buat private key + CSR

```bash
openssl req -new \
  -config etc/root-ca.conf \
  -out ca/root-ca.csr \
  -keyout ca/root-ca/private/root-ca.key
```

### 1.4 Self-sign Root CA certificate

```bash
openssl ca -selfsign \
  -config etc/root-ca.conf \
  -in ca/root-ca.csr \
  -out ca/root-ca.crt \
  -extensions root_ca_ext
```

---

## 🏢 **2. Membuat Signing CA**

### 2.1 Buat direktori

```bash
mkdir -p ca/signing-ca/private ca/signing-ca/db
chmod 700 ca/signing-ca/private
```

### 2.2 Buat file database

```bash
cp /dev/null ca/signing-ca/db/signing-ca.db
echo 01 > ca/signing-ca/db/signing-ca.crt.srl
echo 01 > ca/signing-ca/db/signing-ca.crl.srl
```

### 2.3 Buat private key + CSR

```bash
openssl req -new \
  -config etc/signing-ca.conf \
  -out ca/signing-ca.csr \
  -keyout ca/signing-ca/private/signing-ca.key
```

### 2.4 Root CA menandatangani Signing CA

```bash
openssl ca \
  -config etc/root-ca.conf \
  -in ca/signing-ca.csr \
  -out ca/signing-ca.crt \
  -extensions signing_ca_ext
```

---

## 📧 **3. Mengelola Signing CA**

### 3.1 Buat CSR email untuk Fred

```bash
openssl req -new \
  -config etc/email.conf \
  -out certs/fred.csr \
  -keyout certs/fred.key
```

### 3.2 Terbitkan sertifikat email Fred

```bash
openssl ca \
  -config etc/signing-ca.conf \
  -in certs/fred.csr \
  -out certs/fred.crt \
  -extensions email_ext
```

### 3.3 Buat CSR untuk TLS Server

```bash
SAN=DNS:www.simple.org \
openssl req -new \
  -config etc/server.conf \
  -out certs/simple-org.csr \
  -keyout certs/simple-org.key
```

### 3.4 Terbitkan TLS Server Certificate

```bash
openssl ca \
  -config etc/signing-ca.conf \
  -in certs/simple-org.csr \
  -out certs/simple-org.crt \
  -extensions server_ext
```

### 3.5 Revoke (cabut) sertifikat Fred

> Ganti `29BD8AB...` dengan serial aslinya

```bash
openssl ca \
  -config etc/signing-ca.conf \
  -revoke ca/signing-ca/29BD8AB46D221893E2DFA3F3FBACB2B9A17547DF.pem \
  -crl_reason superseded
```

### 3.6 Buat CRL

```bash
openssl ca -gencrl \
  -config etc/signing-ca.conf \
  -out crl/signing-ca.crl
```

---

## 📤 **4. Format Output Sertifikat**

### 4.1 Convert ke format DER

```bash
openssl x509 \
  -in certs/fred.crt \
  -out certs/fred.cer \
  -outform der
```

### 4.2 Convert CRL ke DER

```bash
openssl crl \
  -in crl/signing-ca.crl \
  -out crl/signing-ca.crl \
  -outform der
```

### 4.3 Buat bundel PKCS#7

```bash
openssl crl2pkcs7 -nocrl \
  -certfile ca/signing-ca.crt \
  -certfile ca/root-ca.crt \
  -out ca/signing-ca-chain.p7c \
  -outform der
```

### 4.4 Buat bundel PKCS#12 (Fred)

```bash
openssl pkcs12 -export \
  -name "Fred Flintstone" \
  -in certs/fred.crt \
  -inkey certs/fred.key \
  -out certs/fred.p12
```

### 4.5 Gabungkan PEM (cert chain dan key + cert)

```bash
cat ca/signing-ca.crt ca/root-ca.crt > ca/signing-ca-chain.pem

cat certs/fred.key certs/fred.crt > certs/fred.pem
```

---

## 🔍 **5. Lihat Isi File Sertifikat/CSR/CRL**

### 5.1 Lihat CSR

```bash
openssl req -in certs/fred.csr -noout -text
```

### 5.2 Lihat Sertifikat

```bash
openssl x509 -in certs/fred.crt -noout -text
```

### 5.3 Lihat CRL

```bash
openssl crl -in crl/signing-ca.crl -inform der -noout -text
```

### 5.4 Lihat isi PKCS#7

```bash
openssl pkcs7 \
  -in ca/signing-ca-chain.p7c \
  -inform der \
  -noout \
  -text \
  -print_certs
```

### 5.5 Lihat isi PKCS#12

```bash
openssl pkcs12 \
  -in certs/fred.p12 \
  -nodes \
  -info
```

---

Semua perintah ini bisa langsung kamu gunakan di terminal selama kamu berada di dalam direktori hasil `git clone` dan sudah tersedia semua file konfigurasi (`etc/*.conf`).

Kalau kamu butuh versi *ringkas* untuk laporan/tugas atau butuh dibantu buatkan file `root-ca.conf`, `signing-ca.conf`, dsb, tinggal bilang.

1. Domain Component         (eg, com)       []:org
2. Domain Component         (eg, company)   []:simple
3. Domain Component         (eg, pki)       []:
4. Organization Name        (eg, company)   []:Simple Inc
5. Organizational Unit Name (eg, section)   []:
6. Common Name              (eg, full name) []:Fred Flintstone
7. Email Address            (eg, name@fqdn) []:fred@simple.org
