# Como Assinar Notas Fiscais com Certificado A3

## Índice
1. [Introdução](#introdução)
2. [Diferenças entre Certificados A1 e A3](#diferenças-entre-certificados-a1-e-a3)
3. [Como Este Projeto Suporta Certificados A3](#como-este-projeto-suporta-certificados-a3)
4. [Implementação Técnica](#implementação-técnica)
5. [Configuração no PHP](#configuração-no-php)
6. [Exemplos de Código](#exemplos-de-código)
7. [Solução de Problemas](#solução-de-problemas)
8. [Referências](#referências)

## Introdução

Este documento explica como o projeto UNINFE permite trabalhar com certificados digitais A3 para assinatura de Notas Fiscais de Serviços Eletrônicas (NFSe), diferentemente de bibliotecas como phpnfe que tradicionalmente suportam apenas certificados A1.

## Diferenças entre Certificados A1 e A3

### Certificado A1
- **Armazenamento**: Arquivo digital (.pfx ou .p12) armazenado no disco rígido
- **Validade**: 1 ano
- **Segurança**: Média - pode ser copiado
- **Mobilidade**: Alta - pode ser transferido entre máquinas
- **Implementação**: Mais simples, leitura direta do arquivo
- **Custo**: Menor

### Certificado A3
- **Armazenamento**: Hardware criptográfico (token USB, cartão inteligente, HSM)
- **Validade**: 1 a 5 anos
- **Segurança**: Alta - não pode ser copiado, a chave privada nunca sai do dispositivo
- **Mobilidade**: Física - requer o dispositivo
- **Implementação**: Requer drivers e bibliotecas específicas
- **Custo**: Maior

## Como Este Projeto Suporta Certificados A3

O projeto UNINFE contém apenas os **arquivos WSDL** (Web Services Description Language) dos serviços de NFSe de diversos municípios brasileiros. Estes arquivos definem:

- Endpoints dos web services
- Estrutura das mensagens XML
- Operações disponíveis (emissão, consulta, cancelamento)
- Esquemas de validação (schemas XSD)

**IMPORTANTE**: Este repositório **não contém código de assinatura digital**. A assinatura é feita pela aplicação PHP que consome estes WSDLs.

## Implementação Técnica

Para trabalhar com certificados A3 em PHP, você precisa:

### 1. Requisitos do Sistema

#### No Linux (Debian/Ubuntu):
```bash
# Instalar bibliotecas OpenSSL e suporte a PKCS#11
sudo apt-get update
sudo apt-get install opensc opensc-pkcs11 libengine-pkcs11-openssl

# Instalar extensões PHP necessárias
sudo apt-get install php-openssl php-soap php-curl php-xml
```

#### No Windows:
```powershell
# Instalar o driver do fabricante do token
# Exemplos: SafeSign, eToken, Certisign, etc.
# Baixar do site do fabricante

# Extensões PHP (geralmente já incluídas)
# php_openssl.dll
# php_soap.dll
# php_curl.dll
```

### 2. Arquitetura da Solução

```
┌─────────────────────────────────────────────────┐
│           Aplicação PHP (Cliente)               │
│  ┌──────────────────────────────────────────┐  │
│  │  1. Gera XML da NFSe (usando WSDL)       │  │
│  │  2. Assina XML com certificado           │  │
│  │  3. Envia para webservice                │  │
│  └──────────────────────────────────────────┘  │
└──────────────┬──────────────────────────────────┘
               │
               ├─── A1: Lê arquivo .pfx direto
               │     └─ openssl_pkcs12_read()
               │
               └─── A3: Acessa via PKCS#11
                     ├─ Token USB / Smartcard
                     ├─ Driver do fabricante
                     └─ Biblioteca PKCS#11
                           │
                           ▼
               ┌────────────────────────┐
               │  Hardware Criptográfico│
               │   (Chave Privada)      │
               └────────────────────────┘
```

### 3. Diferenças na Implementação

#### Com Certificado A1:
```php
// Simples: leitura direta do arquivo
$certificado = file_get_contents('/path/to/certificado.pfx');
$senha = 'senha_do_certificado';

if (openssl_pkcs12_read($certificado, $cert_info, $senha)) {
    $privateKey = $cert_info['pkey'];
    $certificate = $cert_info['cert'];
    // Usa para assinar o XML
}
```

#### Com Certificado A3:
```php
// Complexo: requer acesso ao hardware via PKCS#11

// Opção 1: Usar extensão PKCS#11 PHP (se disponível)
// Opção 2: Usar engine OpenSSL com PKCS#11
// Opção 3: Usar biblioteca externa (ex: chilkat, phpseclib com wrapper)

// Exemplo conceitual (requer configuração adicional):
putenv('PKCS11_MODULE_PATH=/usr/lib/opensc-pkcs11.so');

$tokenConfig = [
    'engine' => 'pkcs11',
    'module' => '/usr/lib/opensc-pkcs11.so',
    'pin' => '1234',  // PIN do token
    'slot' => 0,      // Slot do token
];

// Acesso via engine OpenSSL configurado
$privateKey = openssl_pkey_get_private(
    "pkcs11:token=MeuToken;object=MeuCertificado",
    $tokenConfig
);
```

## Configuração no PHP

### Método Recomendado: Usar Biblioteca NFePHP

A biblioteca **NFePHP** (sucessora do phpnfe) suporta certificados A3 de forma nativa:

```bash
composer require nfephp-org/sped-nfse
```

### Exemplo com NFePHP:

```php
<?php
require_once 'vendor/autoload.php';

use NFePHP\NFSe\Tools;
use NFePHP\Common\Certificate;

// Para certificado A1
$certificadoA1 = Certificate::readPfx(
    file_get_contents('/path/to/certificado.pfx'),
    'senha'
);

// Para certificado A3
// A NFePHP detecta automaticamente se é A3
// e usa a engine PKCS#11 configurada
$certificadoA3 = Certificate::readPfx(
    null,  // não passa conteúdo
    'PIN_DO_TOKEN',
    true,  // indica que é A3
    [
        'tpAmb' => 2,  // Ambiente de homologação
        'pkcs11_engine' => '/usr/lib/engines-1.1/pkcs11.so',
        'pkcs11_module' => '/usr/lib/opensc-pkcs11.so',
    ]
);

// Configurar a Tools com os WSDLs deste repositório
$config = [
    'cnpj' => '12345678000123',
    'im' => '123456',
    'cmun' => '3550308',  // Código do município
    'razao' => 'Razão Social',
    'tpAmb' => 2,
];

$tools = new Tools(json_encode($config), $certificadoA3);

// Usar os WSDLs deste repositório
$tools->loadWsdl('/path/to/uninfe/WSDL/Producao/PSaoPauloSP.wsdl');

// Gerar e assinar a NFSe
$xmlAssinado = $tools->assinarXML($xmlNFSe);

// Enviar para o webservice
$response = $tools->recepcionarLoteRps($xmlAssinado);
```

## Exemplos de Código

### Exemplo 1: Configuração Básica do Token A3

```php
<?php
/**
 * Configuração para uso de certificado A3
 */

// 1. Instalar driver do token (fornecido pelo fabricante)
// 2. Verificar se o token está conectado
// 3. Configurar variáveis de ambiente

// Verificar disponibilidade do módulo PKCS#11
$pkcs11Module = '/usr/lib/opensc-pkcs11.so';  // Linux
// $pkcs11Module = 'C:\\Windows\\System32\\eTPKCS11.dll';  // Windows

if (!file_exists($pkcs11Module)) {
    throw new Exception("Módulo PKCS#11 não encontrado: $pkcs11Module");
}

// Configurar OpenSSL para usar engine PKCS#11
$opensslConfig = <<<CONF
openssl_conf = openssl_def

[openssl_def]
engines = engine_section

[engine_section]
pkcs11 = pkcs11_section

[pkcs11_section]
engine_id = pkcs11
dynamic_path = /usr/lib/engines-1.1/pkcs11.so
MODULE_PATH = $pkcs11Module
init = 0
CONF;

file_put_contents('/tmp/openssl.cnf', $opensslConfig);
putenv('OPENSSL_CONF=/tmp/openssl.cnf');

echo "Configuração A3 concluída!\n";
```

### Exemplo 2: Assinatura XML com A3

```php
<?php
/**
 * Assinar XML usando certificado A3
 */

class AssinadorA3 {
    private $pin;
    private $slotId;
    
    public function __construct($pin, $slotId = 0) {
        $this->pin = $pin;
        $this->slotId = $slotId;
    }
    
    public function assinarXML($xmlString) {
        // Carregar o XML
        $dom = new DOMDocument('1.0', 'UTF-8');
        $dom->preserveWhiteSpace = false;
        $dom->formatOutput = false;
        $dom->loadXML($xmlString);
        
        // Criar a assinatura usando XMLSecLibs
        $objDSig = new XMLSecurityDSig();
        $objDSig->setCanonicalMethod(XMLSecurityDSig::EXC_C14N);
        
        // Adicionar referência ao nó a ser assinado
        $objDSig->addReference(
            $dom,
            XMLSecurityDSig::SHA1,
            ['http://www.w3.org/2000/09/xmldsig#enveloped-signature'],
            ['force_uri' => true]
        );
        
        // Criar nova chave
        $objKey = new XMLSecurityKey(
            XMLSecurityKey::RSA_SHA1,
            ['type' => 'private']
        );
        
        // Configurar para usar token A3
        $objKey->loadKey("pkcs11:", false, false);
        
        // Assinar
        $objDSig->sign($objKey);
        
        // Adicionar certificado
        $objDSig->add509Cert($this->getCertificadoDoToken());
        
        // Inserir assinatura no XML
        $objDSig->appendSignature($dom->documentElement);
        
        return $dom->saveXML();
    }
    
    private function getCertificadoDoToken() {
        // Extrair certificado do token
        // Implementação específica do driver
        // Retorna o certificado em formato PEM
    }
}

// Uso
$assinador = new AssinadorA3('1234');  // PIN do token
$xmlAssinado = $assinador->assinarXML($xmlNFSe);
```

### Exemplo 3: Integração Completa com WSDLs do UNINFE

```php
<?php
/**
 * Exemplo completo de uso dos WSDLs do UNINFE com certificado A3
 */

require_once 'vendor/autoload.php';

use NFePHP\NFSe\Tools;
use NFePHP\Common\Certificate;

class GeradorNFSeA3 {
    private $tools;
    private $municipio;
    
    public function __construct($config, $pinToken) {
        // Configurar certificado A3
        $certificate = Certificate::readPfx(
            null,
            $pinToken,
            true,
            [
                'pkcs11_engine' => '/usr/lib/engines-1.1/pkcs11.so',
                'pkcs11_module' => '/usr/lib/opensc-pkcs11.so',
            ]
        );
        
        // Inicializar Tools com configuração
        $this->tools = new Tools(json_encode($config), $certificate);
        $this->municipio = $config['cmun'];
        
        // Carregar WSDL apropriado do repositório UNINFE
        $this->carregarWSDL($this->municipio);
    }
    
    private function carregarWSDL($codigoMunicipio) {
        // Mapear código do município para WSDL
        $mapeamento = [
            '3550308' => 'PSaoPauloSP.wsdl',  // São Paulo
            '3304557' => 'PRioDeJaneiroRJ.wsdl',  // Rio de Janeiro
            '4106902' => 'PCuritibaPR.wsdl',  // Curitiba
            // ... outros municípios
        ];
        
        $wsdlFile = $mapeamento[$codigoMunicipio] ?? null;
        
        if (!$wsdlFile) {
            throw new Exception("WSDL não encontrado para município: $codigoMunicipio");
        }
        
        $wsdlPath = __DIR__ . "/WSDL/Producao/$wsdlFile";
        
        if (!file_exists($wsdlPath)) {
            throw new Exception("Arquivo WSDL não encontrado: $wsdlPath");
        }
        
        $this->tools->loadWsdl($wsdlPath);
    }
    
    public function emitirNFSe($dadosNFSe) {
        // Construir XML da NFSe
        $xml = $this->construirXML($dadosNFSe);
        
        // Assinar com certificado A3 (feito automaticamente pela Tools)
        $response = $this->tools->recepcionarLoteRps($xml);
        
        return $response;
    }
    
    private function construirXML($dados) {
        // Construir XML conforme layout do município
        // Usar schemas do repositório UNINFE para validação
        $xml = '<?xml version="1.0" encoding="UTF-8"?>';
        $xml .= '<EnviarLoteRpsEnvio>';
        // ... construir XML conforme schema
        $xml .= '</EnviarLoteRpsEnvio>';
        
        return $xml;
    }
}

// Configuração
$config = [
    'cnpj' => '12345678000123',
    'im' => '123456',
    'cmun' => '3550308',  // São Paulo
    'razao' => 'Minha Empresa LTDA',
    'tpAmb' => 2,  // Homologação
];

// Criar gerador com certificado A3
$gerador = new GeradorNFSeA3($config, '1234');  // PIN do token

// Emitir NFSe
$dadosNFSe = [
    'numero' => 1,
    'valor' => 1000.00,
    'tomador' => [
        'cnpj' => '98765432000198',
        'razao' => 'Cliente LTDA',
    ],
    // ... outros dados
];

try {
    $response = $gerador->emitirNFSe($dadosNFSe);
    echo "NFSe emitida com sucesso!\n";
    print_r($response);
} catch (Exception $e) {
    echo "Erro ao emitir NFSe: " . $e->getMessage() . "\n";
}
```

## Solução de Problemas

### Problema 1: Token não detectado

**Sintomas**: Erro "Token não encontrado" ou "PKCS#11 module error"

**Soluções**:
```bash
# Verificar se o token está conectado (Linux)
pkcs11-tool --list-slots

# Verificar módulos disponíveis
pkcs11-tool --list-modules

# Testar acesso ao token
pkcs11-tool --login --test --pin 1234
```

### Problema 2: PIN incorreto

**Sintomas**: Erro "CKR_PIN_INCORRECT" ou "Invalid PIN"

**Soluções**:
- Verificar se o PIN está correto
- Atenção: 3 tentativas incorretas podem bloquear o token
- Usar o PIN de desbloqueio (PUK) se bloqueado

### Problema 3: Certificado expirado

**Sintomas**: Erro de validação de certificado

**Soluções**:
```php
// Verificar validade do certificado
$certData = openssl_x509_parse($certificate);
$validFrom = $certData['validFrom_time_t'];
$validTo = $certData['validTo_time_t'];
$now = time();

if ($now < $validFrom || $now > $validTo) {
    echo "Certificado expirado ou ainda não válido\n";
    echo "Válido de: " . date('d/m/Y', $validFrom) . "\n";
    echo "Válido até: " . date('d/m/Y', $validTo) . "\n";
}
```

### Problema 4: Biblioteca PKCS#11 não encontrada

**Sintomas**: Erro "Failed to load PKCS#11 module"

**Soluções**:
```bash
# Ubuntu/Debian
sudo apt-get install opensc opensc-pkcs11

# CentOS/RHEL
sudo yum install opensc

# Verificar localização
find /usr -name "opensc-pkcs11.so" 2>/dev/null
find /usr -name "libpkcs11.so" 2>/dev/null
```

### Problema 5: Permissões de acesso ao dispositivo

**Sintomas**: Erro "Permission denied" ao acessar token

**Soluções**:
```bash
# Adicionar usuário ao grupo apropriado (Linux)
sudo usermod -a -G pcscd $USER

# Reiniciar serviço pcscd
sudo systemctl restart pcscd

# Verificar permissões do dispositivo
ls -l /dev/bus/usb/
```

## Por que Este Projeto (UNINFE) Funciona com A3

A razão pela qual este projeto suporta certificados A3 **não está no repositório em si**, mas sim em:

1. **Independência**: Os WSDLs são apenas definições de serviços. Eles não fazem suposições sobre como o XML será assinado.

2. **Compatibilidade**: Os schemas XSD incluídos validam apenas a estrutura do XML, não o método de assinatura.

3. **Padrão W3C**: A assinatura XML segue o padrão XML Digital Signature (XMLDSig), que é agnóstico em relação ao tipo de certificado.

4. **Implementação no cliente**: A aplicação PHP que usa estes WSDLs é responsável por:
   - Acessar o certificado (A1 ou A3)
   - Assinar o XML
   - Enviar para o webservice

### Comparação com phpnfe

O **phpnfe** tradicional tinha limitação para A3 porque:
- Assumia acesso direto ao arquivo de certificado
- Não tinha suporte para engine PKCS#11
- Não abstraía o acesso ao hardware criptográfico

A **NFePHP** (versão moderna) resolve isso:
- Suporta múltiplos métodos de acesso ao certificado
- Integra-se com engines OpenSSL
- Abstrai a complexidade do PKCS#11

### Fluxo Técnico Completo

```
┌─────────────────────────────────────────────────────────┐
│                    Aplicação PHP                         │
│                                                          │
│  1. Carrega WSDL do UNINFE                              │
│     └─ Define estrutura e endpoint do serviço           │
│                                                          │
│  2. Constrói XML da NFSe                                │
│     └─ Valida com schemas XSD do UNINFE                 │
│                                                          │
│  3. Assina XML                                          │
│     ├─ A1: openssl_pkcs12_read()                        │
│     └─ A3: PKCS#11 engine                               │
│         ├─ Acessa token via USB                         │
│         ├─ Envia dados para assinar                     │
│         ├─ Token assina internamente                    │
│         └─ Retorna assinatura                           │
│                                                          │
│  4. Envia XML assinado via SOAP                         │
│     └─ Endpoint definido no WSDL                        │
│                                                          │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
            ┌──────────────────────────┐
            │   Webservice Municipal    │
            │   (Prefeitura)           │
            └──────────────────────────┘
```

## Configuração de Ambiente de Desenvolvimento

### Docker com Suporte a A3

```dockerfile
FROM php:8.2-apache

# Instalar dependências
RUN apt-get update && apt-get install -y \
    opensc \
    opensc-pkcs11 \
    libengine-pkcs11-openssl \
    pcscd \
    pcsc-tools \
    libpcsclite-dev \
    && docker-php-ext-install soap \
    && a2enmod rewrite

# Copiar WSDLs do UNINFE
COPY ./WSDL /var/www/html/wsdl/
COPY ./schemas /var/www/html/schemas/

# Configurar OpenSSL
RUN echo "openssl_conf = openssl_def\n\
[openssl_def]\n\
engines = engine_section\n\
[engine_section]\n\
pkcs11 = pkcs11_section\n\
[pkcs11_section]\n\
engine_id = pkcs11\n\
dynamic_path = /usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so\n\
MODULE_PATH = /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so\n\
init = 0" > /etc/ssl/openssl.cnf

# Iniciar serviço pcscd
CMD service pcscd start && apache2-foreground
```

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  nfse-app:
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./app:/var/www/html/app
      - ./WSDL:/var/www/html/wsdl
      - ./schemas:/var/www/html/schemas
    devices:
      - /dev/bus/usb:/dev/bus/usb  # Acesso ao token USB
    privileged: true
    environment:
      - PKCS11_MODULE_PATH=/usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
```

### Teste de Conectividade com Token

```php
<?php
/**
 * Script de teste para verificar conectividade com token A3
 */

echo "=== Teste de Token A3 ===\n\n";

// 1. Verificar módulo PKCS#11
$modulePath = getenv('PKCS11_MODULE_PATH') ?: '/usr/lib/opensc-pkcs11.so';
echo "1. Verificando módulo PKCS#11...\n";
echo "   Caminho: $modulePath\n";
echo "   Existe: " . (file_exists($modulePath) ? "SIM" : "NÃO") . "\n\n";

// 2. Verificar serviço pcscd
echo "2. Verificando serviço pcscd...\n";
$pcscdStatus = shell_exec('service pcscd status 2>&1');
echo "   Status: " . (strpos($pcscdStatus, 'running') !== false ? "ATIVO" : "INATIVO") . "\n\n";

// 3. Listar tokens disponíveis
echo "3. Listando tokens disponíveis...\n";
$slots = shell_exec('pkcs11-tool --list-slots 2>&1');
echo $slots . "\n";

// 4. Testar abertura de sessão (requer PIN)
$pin = readline("Digite o PIN do token (ou Enter para pular): ");
if (!empty($pin)) {
    echo "\n4. Testando autenticação...\n";
    $auth = shell_exec("pkcs11-tool --login --test --pin $pin 2>&1");
    echo $auth . "\n";
    
    // 5. Listar certificados
    echo "\n5. Listando certificados no token...\n";
    $certs = shell_exec("pkcs11-tool --login --list-objects --type cert --pin $pin 2>&1");
    echo $certs . "\n";
}

echo "\n=== Teste concluído ===\n";
```

## Melhores Práticas

### 1. Segurança

```php
// ❌ NÃO FAZER: Hardcoded PIN
$pin = '1234';

// ✅ FAZER: PIN em variável de ambiente
$pin = getenv('TOKEN_PIN');

// ✅ FAZER: PIN em arquivo de configuração protegido
$config = parse_ini_file('/etc/nfse/config.ini');
$pin = $config['token_pin'];
```

### 2. Tratamento de Erros

```php
try {
    $certificate = Certificate::readPfx(null, $pin, true, $pkcs11Config);
} catch (Exception $e) {
    // Log detalhado
    error_log('Erro ao acessar certificado A3: ' . $e->getMessage());
    error_log('Stack trace: ' . $e->getTraceAsString());
    
    // Mensagem amigável ao usuário
    if (strpos($e->getMessage(), 'CKR_PIN_INCORRECT') !== false) {
        throw new Exception('PIN incorreto. Tentativas restantes: X');
    } elseif (strpos($e->getMessage(), 'token not found') !== false) {
        throw new Exception('Token não detectado. Verifique a conexão USB.');
    } else {
        throw new Exception('Erro ao acessar certificado: ' . $e->getMessage());
    }
}
```

### 3. Cache de Sessão

```php
// Manter sessão aberta para múltiplas assinaturas
class TokenSession {
    private static $session;
    
    public static function getSession($pin) {
        if (self::$session === null) {
            self::$session = self::openSession($pin);
        }
        return self::$session;
    }
    
    private static function openSession($pin) {
        // Abrir sessão com token
        // Retornar handle da sessão
    }
    
    public static function closeSession() {
        if (self::$session !== null) {
            // Fechar sessão
            self::$session = null;
        }
    }
}

// Uso
$session = TokenSession::getSession($pin);
// ... múltiplas operações ...
TokenSession::closeSession();
```

### 4. Validação de Certificado

```php
function validarCertificado($certificate) {
    $certData = openssl_x509_parse($certificate);
    
    // Verificar validade temporal
    $now = time();
    if ($now < $certData['validFrom_time_t']) {
        throw new Exception('Certificado ainda não é válido');
    }
    if ($now > $certData['validTo_time_t']) {
        throw new Exception('Certificado expirado');
    }
    
    // Verificar tipo (e-CNPJ ou e-CPF)
    $subject = $certData['subject'];
    if (!isset($subject['CN'])) {
        throw new Exception('Certificado inválido: CN não encontrado');
    }
    
    // Avisar se próximo da expiração (30 dias)
    $diasRestantes = ($certData['validTo_time_t'] - $now) / 86400;
    if ($diasRestantes < 30) {
        error_log("Aviso: Certificado expira em $diasRestantes dias");
    }
    
    return true;
}
```

## Referências

### Documentação Oficial
- [NFePHP - Documentação](https://nfephp.org)
- [ABRASF - Padrão Nacional NFSe](http://www.abrasf.org.br/nfse)
- [OpenSC - Smart Card Tools](https://github.com/OpenSC/OpenSC/wiki)
- [PKCS#11 Specification](http://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/os/pkcs11-base-v2.40-os.html)

### Bibliotecas Recomendadas
- **NFePHP**: https://github.com/nfephp-org/sped-nfse
- **XMLSecLibs**: https://github.com/robrichards/xmlseclibs
- **phpseclib**: https://github.com/phpseclib/phpseclib

### Drivers de Token
- **SafeSign**: https://www.aeteurope.com/
- **eToken (Safenet)**: https://safenet.gemalto.com/
- **Certisign**: https://www.certisign.com.br/
- **Serasa**: https://serasa.certificadodigital.com.br/

### Comunidade
- [Fórum NFePHP](https://github.com/nfephp-org/sped-nfse/discussions)
- [Stack Overflow - NFSe Brasil](https://stackoverflow.com/questions/tagged/nfse)

## Conclusão

Este repositório UNINFE fornece os **WSDLs e schemas necessários** para comunicação com os webservices municipais de NFSe. A capacidade de usar certificados A3 não está nos WSDLs em si, mas sim na **implementação do cliente PHP** que:

1. Usa os WSDLs para conhecer a estrutura dos serviços
2. Constrói os XMLs conforme os schemas
3. **Assina os XMLs** usando certificado A3 via PKCS#11
4. Envia para os endpoints definidos nos WSDLs

A diferença fundamental entre A1 e A3 está na **camada de assinatura digital**, que ocorre antes do envio ao webservice. Os WSDLs são neutros quanto a isso.

**Dica Final**: Para migrar de phpnfe com A1 para NFePHP com A3, você precisa:
1. Instalar os drivers do token A3
2. Configurar o OpenSSL com engine PKCS#11
3. Atualizar o código para usar NFePHP
4. Usar os WSDLs deste repositório
5. Testar em ambiente de homologação

---

**Contribuições**: Se você encontrou uma forma melhor de trabalhar com A3, por favor contribua com este documento!

**Licença**: Este documento segue a mesma licença do projeto UNINFE.
