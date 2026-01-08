# UNINFE - Projeto Documentos Fiscais Eletrônicos - NFSe

Mirror do projeto original: http://sourceforge.net/p/uninfe/code

## Sobre o Projeto

Este repositório contém **WSDLs (Web Services Description Language)** e **schemas XSD** para integração com os webservices de **NFSe (Nota Fiscal de Serviços Eletrônica)** de diversos municípios brasileiros.

### Conteúdo

- **WSDL/Producao/**: WSDLs para ambientes de produção
- **WSDL/Homologacao/**: WSDLs para ambientes de teste/homologação
- **schemas/**: Schemas XSD para validação dos XMLs
- **Webservice.xml**: Mapeamento de municípios e seus respectivos WSDLs

## 📜 Certificados Digitais A1 e A3

Uma das principais dúvidas sobre NFSe é como trabalhar com diferentes tipos de certificados digitais.

### 📖 [Guia Completo: Assinatura com Certificados A3](./CERTIFICADOS_A3.md)

Este projeto suporta tanto certificados **A1** (arquivo) quanto **A3** (token/smartcard). Veja o guia completo para entender:

- ✅ Diferenças entre certificados A1 e A3
- ✅ Como este projeto permite usar A3 (diferente do phpnfe tradicional)
- ✅ Implementação técnica com PHP
- ✅ Exemplos de código prontos para uso
- ✅ Configuração de ambiente (Docker, Linux, Windows)
- ✅ Solução de problemas comuns
- ✅ Melhores práticas de segurança

**[👉 Acesse o guia completo de certificados A3](./CERTIFICADOS_A3.md)**

## Como Usar

### 1. Identificar o Município

Consulte o arquivo `WSDL/Webservice.xml` para encontrar o código IBGE do seu município e o padrão de webservice utilizado.

### 2. Selecionar o WSDL Apropriado

Escolha o WSDL correspondente ao município:
- Produção: `WSDL/Producao/P{Municipio}.wsdl`
- Homologação: `WSDL/Homologacao/H{Municipio}.wsdl`

### 3. Integrar com sua Aplicação

```php
// Exemplo básico com PHP
$wsdl = 'WSDL/Producao/PSaoPauloSP.wsdl';
$client = new SoapClient($wsdl, [
    'local_cert' => '/path/to/certificado.pem',
    'passphrase' => 'senha_certificado'
]);

// Para certificado A3, veja o guia completo
```

## Padrões Suportados

O projeto inclui WSDLs para diversos padrões de webservices, incluindo:

- **GINFES** - Padrão nacional ABRASF
- **BETHA**
- **IPM**
- **ISSNET**
- **TIPLAN**
- **FIORILLI**
- **PRONIN**
- **SIMPLISS**
- E muitos outros...

## Bibliotecas Recomendadas

Para facilitar a integração, recomendamos o uso de bibliotecas especializadas:

### NFePHP (Recomendado)
```bash
composer require nfephp-org/sped-nfse
```

A **NFePHP** oferece:
- ✅ Suporte nativo a certificados A1 e A3
- ✅ Abstração dos diferentes padrões municipais
- ✅ Validação automática com schemas XSD
- ✅ Assinatura digital integrada
- ✅ Comunidade ativa e documentação completa

[Documentação NFePHP](https://nfephp.org)

## Estrutura do Repositório

```
uninfe/
├── WSDL/
│   ├── Producao/          # WSDLs de produção
│   ├── Homologacao/       # WSDLs de homologação
│   └── Webservice.xml     # Mapeamento de municípios
├── schemas/
│   └── NFSe/              # Schemas XSD por padrão
│       ├── ABRASF/
│       ├── GINFES/
│       ├── BETHA/
│       └── ...
├── CERTIFICADOS_A3.md     # Guia completo sobre certificados
└── README.md              # Este arquivo
```

## Contribuindo

Este é um mirror do projeto original do SourceForge. Para contribuições, consulte o projeto original.

## Origem do Repositório

```bash
# Clone original do SVN
git svn clone http://svn.code.sf.net/p/uninfe/code --no-metadata --stdlayout uninfe-git
git filter-branch --subdirectory-filter fontes/NFe.Components.Wsdl/NFse
```

## Links Úteis

- [Projeto Original no SourceForge](http://sourceforge.net/p/uninfe/code)
- [NFePHP - Biblioteca PHP](https://github.com/nfephp-org/sped-nfse)
- [ABRASF - Associação Brasileira das Secretarias de Finanças](http://www.abrasf.org.br/)
- [Portal Nacional da NFSe](https://www.nfse.gov.br/)

## Suporte

Para questões sobre:
- **WSDLs e schemas**: Use as issues deste repositório
- **Implementação PHP**: Consulte a [comunidade NFePHP](https://github.com/nfephp-org/sped-nfse/discussions)
- **Certificados A3**: Leia o [guia completo](./CERTIFICADOS_A3.md)

## Licença

Este projeto mantém a licença original do projeto UNINFE no SourceForge.
