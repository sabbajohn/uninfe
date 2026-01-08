# Understanding A3 Certificate Support in Brazilian NFSe

## Quick Summary

**Question**: How does this project support A3 certificates (hardware tokens) when libraries like phpnfe only work with A1 certificates (files)?

**Answer**: This repository contains **WSDL files** (Web Service definitions) which are independent of certificate handling. The certificate signing happens in your PHP application, not in the WSDLs. Modern libraries like NFePHP support both A1 and A3 certificates.

## Background: Brazilian Digital Certificates

In Brazil, electronic invoices (NFe/NFSe) must be digitally signed using ICP-Brasil certificates. There are two main types:

### A1 Certificates (Software)
- Stored as files (.pfx, .p12)
- Valid for 1 year
- Can be copied/backed up
- Simple to implement: read file → sign
- Lower cost

### A3 Certificates (Hardware)
- Stored in cryptographic hardware (USB token, smart card, HSM)
- Valid for 1-5 years
- Cannot be copied (private key never leaves device)
- More secure but complex to implement
- Higher cost
- Required by some Brazilian municipalities for certain businesses

## Architecture

```
┌─────────────────────────────────┐
│     PHP Application             │
│  ┌──────────────────────────┐  │
│  │ 1. Load WSDL from UNINFE │  │
│  │ 2. Build XML             │  │
│  │ 3. Sign XML (A1 or A3)   │  │
│  │ 4. Send to webservice    │  │
│  └──────────────────────────┘  │
└──────────────┬──────────────────┘
               │
               ├─── A1: openssl_pkcs12_read()
               │
               └─── A3: PKCS#11 engine
                     ├─ USB Token
                     ├─ Driver
                     └─ PKCS#11 library
```

## Why UNINFE Works with Both

The UNINFE repository contains:
- ✅ WSDL files (service definitions)
- ✅ XSD schemas (XML validation)
- ✅ Endpoint mappings

It does NOT contain:
- ❌ Certificate handling code
- ❌ XML signing code
- ❌ HTTP client code

**Key Insight**: WSDLs are certificate-agnostic. They define:
- What operations are available (submit, query, cancel)
- What the XML structure should be
- Where to send the requests

The actual certificate usage happens in your PHP code, which can use either A1 or A3.

## Implementation Difference

### With A1 (Simple):
```php
$pfx = file_get_contents('/path/to/cert.pfx');
openssl_pkcs12_read($pfx, $cert, 'password');
// Use $cert['pkey'] and $cert['cert'] to sign XML
```

### With A3 (Complex):
```php
// Requires:
// - Token driver installed
// - PKCS#11 library
// - OpenSSL engine configuration
// - PIN for token

$privateKey = openssl_pkey_get_private(
    "pkcs11:token=MyToken;object=MyCert",
    ['engine' => 'pkcs11', 'pin' => '1234']
);
// Use $privateKey to sign XML
```

## Recommended Solution: NFePHP

The modern **NFePHP** library handles both A1 and A3 seamlessly:

```bash
composer require nfephp-org/sped-nfse
```

```php
use NFePHP\NFSe\Tools;
use NFePHP\Common\Certificate;

// For A3, NFePHP handles PKCS#11 automatically
$certificate = Certificate::readPfx(
    null,  // no file content for A3
    'PIN',
    true,  // indicates A3
    [
        'pkcs11_engine' => '/path/to/pkcs11.so',
        'pkcs11_module' => '/path/to/opensc-pkcs11.so',
    ]
);

$tools = new Tools(json_encode($config), $certificate);
$tools->loadWsdl('/path/to/uninfe/WSDL/Producao/PCity.wsdl');
```

## System Requirements for A3

### Linux (Debian/Ubuntu):
```bash
sudo apt-get install opensc opensc-pkcs11 libengine-pkcs11-openssl
sudo apt-get install php-openssl php-soap
```

### Windows:
- Install token driver from manufacturer
- PHP extensions: php_openssl.dll, php_soap.dll

## Common Issues

### 1. Token Not Detected
```bash
# Test token connection
pkcs11-tool --list-slots
```

### 2. Wrong PIN
- 3 incorrect attempts will lock the token
- Use PUK (unblock PIN) to unlock

### 3. Certificate Expired
```php
$certData = openssl_x509_parse($certificate);
$expiryDate = date('Y-m-d', $certData['validTo_time_t']);
```

## Docker Example

```dockerfile
FROM php:8.2-apache

RUN apt-get update && apt-get install -y \
    opensc opensc-pkcs11 pcscd \
    && docker-php-ext-install soap

# Copy UNINFE WSDLs
COPY ./WSDL /var/www/html/wsdl/
COPY ./schemas /var/www/html/schemas/

# Allow USB device access
# Run with: docker run --device=/dev/bus/usb:/dev/bus/usb
```

## Security Best Practices

```php
// ❌ DON'T: Hardcode PIN
$pin = '1234';

// ✅ DO: Use environment variable
$pin = getenv('TOKEN_PIN');

// ✅ DO: Use secure config file
$config = parse_ini_file('/secure/path/config.ini');
$pin = $config['token_pin'];
```

## Why phpnfe Doesn't Support A3

Traditional **phpnfe** had limitations because:
1. It assumed direct file access to certificates
2. No PKCS#11 engine integration
3. No abstraction for hardware cryptography

**NFePHP** (the successor) fixed this by:
1. Abstracting certificate access
2. Supporting OpenSSL engines
3. Handling PKCS#11 complexity internally

## Complete Example

```php
<?php
require 'vendor/autoload.php';

use NFePHP\NFSe\Tools;
use NFePHP\Common\Certificate;

// Configuration
$config = [
    'cnpj' => '12345678000123',
    'im' => '123456',
    'cmun' => '3550308',  // São Paulo
    'razao' => 'My Company Ltd',
    'tpAmb' => 2,  // Test environment
];

// A3 Certificate
$certificate = Certificate::readPfx(
    null,
    '1234',  // PIN - use env var in production
    true,
    [
        'pkcs11_engine' => '/usr/lib/engines-1.1/pkcs11.so',
        'pkcs11_module' => '/usr/lib/opensc-pkcs11.so',
    ]
);

// Initialize with UNINFE WSDL
$tools = new Tools(json_encode($config), $certificate);
$tools->loadWsdl(__DIR__ . '/WSDL/Producao/PSaoPauloSP.wsdl');

// Build and send invoice
$xml = buildInvoiceXML($data);  // Your function
$response = $tools->recepcionarLoteRps($xml);

print_r($response);
```

## Key Takeaways

1. **UNINFE = WSDLs only**: No certificate code
2. **WSDLs are certificate-agnostic**: Work with any signing method
3. **A3 requires PKCS#11**: Hardware token access library
4. **Use NFePHP**: Handles A1 and A3 complexity
5. **Security**: Never hardcode PINs, use env vars

## Resources

- [NFePHP GitHub](https://github.com/nfephp-org/sped-nfse)
- [PKCS#11 Specification](http://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/)
- [OpenSC Tools](https://github.com/OpenSC/OpenSC)
- [ICP-Brasil](https://www.gov.br/iti/pt-br/assuntos/icp-brasil)

## Full Portuguese Documentation

For complete documentation in Portuguese including detailed code examples, troubleshooting, and best practices, see:

**[📖 CERTIFICADOS_A3.md](./CERTIFICADOS_A3.md)** (814 lines, comprehensive guide)

---

**Note**: This is a simplified English overview. Brazilian developers should refer to the Portuguese documentation for complete details, as Brazilian tax regulations and certificate authorities are complex and region-specific.
