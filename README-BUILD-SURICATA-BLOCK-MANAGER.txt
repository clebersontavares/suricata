BLOCK MANAGER - BUILD DO MOTOR SURICATA
========================================

Documento tecnico do processo utilizado para compilar e empacotar o Suricata 8.0.8 como motor isolado do Block Manager para FreeBSD 16 amd64 / pfSense 2.9.

1. OBJETIVO
-----------

Criar um pacote nativo FreeBSD contendo somente o motor Suricata e suas ferramentas, sem instalar a WebGUI tradicional do pacote pfSense Suricata e sem alterar PF ou iniciar o motor automaticamente.

Pacote produzido:
  blockmanager-suricata-engine-8.0.8_1.pkg

ABI:
  FreeBSD:16:amd64

SHA-256 do pacote:
  7a987e3ff7b3475cb6cbd2fbe4f6af60fc008791a10bb8be0cfed0b4554d278f

2. AMBIENTE DE BUILD
-------------------

Sistema:
  FreeBSD 16.0-CURRENT
  amd64
  ABI FreeBSD:16:amd64
  Kernel ABI observado: 1600031

Recursos utilizados:
  1 vCPU
  2 GB RAM
  2 GB swap
  aproximadamente 14 GB livres antes do build

A compilacao foi executada serialmente para reduzir consumo de memoria.

Variaveis utilizadas:
  MAKE_JOBS_NUMBER=1
  CARGO_BUILD_JOBS=1
  CARGO_INCREMENTAL=0
  TMPDIR=/root/blockmanager-build/suricata/tmp
  CARGO_HOME=/root/blockmanager-build/suricata/cargo
  CARGO_NET_RETRY=5

3. FONTE
--------

Versao:
  Suricata 8.0.8 RELEASE

Fonte oficial:
  https://www.openinfosecfoundation.org/download/suricata-8.0.8.tar.gz

Assinatura disponibilizada pelo projeto:
  https://www.openinfosecfoundation.org/download/suricata-8.0.8.tar.gz.sig

SHA-256 observado no tarball utilizado:
  bb56bbaeb4da81b4c90e61148f93b789ea924412207388bc9cba0a7ac6a24a8f

O build nunca utiliza a branch main diretamente. A versao e o tarball ficam fixados para permitir reproducao.

4. DEPENDENCIAS DE BUILD/RUNTIME
--------------------------------

Pacotes confirmados no ambiente de build:

  rust             1.99.1_1
  rust-cbindgen    0.29.4_4,1
  python312        3.12.16
  py312-pyyaml     6.0.3_1
  jansson          2.15.1_1
  libyaml          0.2.5
  pcre2            10.49
  libnet           1.3,1
  libmaxminddb     1.14.1
  liblz4           1.10.0_2,1
  hyperscan        5.4.2
  gmake            4.4.1
  autoconf         2.73
  automake         1.19
  libtool          2.6.2
  pkgconf          3.0.9

Headers do sistema confirmados:
  /usr/include/pcap.h
  /usr/include/pcap/pcap.h
  /usr/include/net/netmap_user.h
  /usr/include/net/netmap.h

Dependencias declaradas no pacote do motor:
  python312
  py312-pyyaml
  liblz4
  libmaxminddb
  libnet
  jansson
  libyaml
  pcre2
  hyperscan

Bibliotecas-base do FreeBSD sao usadas do proprio sistema, incluindo libpcap, libnetmap, libmagic, libelf, libc e libthr.

5. ESTRUTURA DE BUILD
---------------------

/root/blockmanager-build/suricata/
  source/      tarball e assinatura originais
  build/       fonte extraida e objetos compilados
  stage/       install-full usando DESTDIR
  pkgroot/     arvore preparada para pkg create
  package/     +MANIFEST e pkg-plist
  packages/    pacotes finais
  logs/        configure, build, staging, hashes e validacoes
  tmp/         temporarios
  cargo/       CARGO_HOME isolado

6. CONFIGURE
------------

O configure final foi executado com prefixo isolado do Block Manager:

  --prefix=/usr/local/blockmanager/suricata
  --sysconfdir=/usr/local/etc/blockmanager
  --localstatedir=/var
  --disable-gccmarch-native
  --enable-netmap
  --enable-geoip

Variaveis de compilacao:
  CPPFLAGS=-I/usr/local/include
  LDFLAGS=-L/usr/local/lib -Wl,-rpath,/usr/local/blockmanager/suricata/lib -Wl,-rpath,/usr/local/lib
  PKG_CONFIG_PATH=/usr/local/libdata/pkgconfig:/usr/local/share/pkgconfig

Foi criado /usr/local/bin/python3 apontando para Python 3.12 porque o configure procura python3 pelo PATH.

Resultado do configure:
  Netmap support:        yes v14+
  GeoIP2 support:        yes
  Hyperscan support:     yes
  Rust:                  yes
  Python:                yes
  suricata-update:       yes
  suricatactl:           yes
  suricatasc:            yes
  GCC march native:      no

CFLAGS finais:
  -g -O2 -DOS_FREEBSD -std=c11

-march=native foi explicitamente desativado para evitar gerar um binario preso a CPU da maquina de build.

7. COMPILACAO
-------------

Comando principal:
  gmake -j1

A compilacao foi executada em background com nohup e codigo de saida gravado em arquivo, para sobreviver a queda da sessao SSH.

Resultado:
  build.exit = 0

Uso observado de swap ao final:
  aproximadamente 19 MB de 2 GB

8. STAGING
----------

O motor nao foi instalado diretamente no sistema durante o build.

Foi usado:
  gmake DESTDIR=/root/blockmanager-build/suricata/stage install-full

Resultado:
  stage_exit=0
  staging aproximadamente 65 MB
  123 arquivos antes da limpeza de caches Python

O staging confirmou:
  suricata
  suricata-update
  suricatactl
  suricatasc
  modulos Python do suricata-update
  configuracao-base
  regras internas de eventos
  documentacao

__pycache__ e arquivos .pyc foram removidos do pkgroot antes do empacotamento.

9. RECURSOS VALIDADOS
---------------------

suricata --build-info confirmou:
  Suricata 8.0.8 RELEASE
  Netmap v14+
  GeoIP2
  Hyperscan
  JA3
  JA4
  Rust
  LZ4

Recursos QUIC confirmados:
  protocolo de aplicacao quic
  quic.version
  quic.sni
  quic.ua
  quic.cyu.hash
  quic.cyu.string

GeoIP confirmado:
  keyword geoip

Datasets confirmados:
  keyword dataset

Componentes compilados observados incluem:
  detect-quic-sni.o
  detect-quic-version.o
  detect-quic-ua.o
  detect-geoip.o
  output-eve.o
  runmode-netmap.o

Tambem foram instaladas regras internas como:
  quic-events.rules
  ipsec-events.rules

10. VALIDACAO DE CONFIGURACAO
-----------------------------

Um teste inicial em DESTDIR falhou porque suricata.yaml contem caminhos absolutos finais. Isso nao era falha de compilacao.

Depois de disponibilizar classification.config, reference.config, threshold.config e suricata.yaml nos caminhos finais de teste, foi usada uma regra minima:

  alert udp any any -> any 443 (msg:"Block Manager UDP 443 test"; sid:9000001; rev:1;)

Teste:
  suricata -T -c /usr/local/etc/blockmanager/suricata/suricata.yaml -S /var/lib/suricata/rules/blockmanager-test.rules -l /var/log/suricata

Resultado:
  Configuration provided was successfully loaded.
  test_exit=0

O EVE JSON, fast.log e stats.log foram inicializados no teste.

11. SURICATA-UPDATE
-------------------

Versao validada:
  suricata-update 1.3.8

Fontes observadas no indice integrado incluem:
  et/open
  et/pro
  oisf/trafficid
  abuse.ch/feodotracker
  abuse.ch/sslbl-blacklist
  abuse.ch/sslbl-ja3
  abuse.ch/urlhaus
  stamus/lateral
  outras fontes gratuitas e comerciais

Modelo previsto no Block Manager:
  ET Open como baseline gratuita
  ET Pro opcional para cliente com assinatura
  OISF TrafficID e feeds abuse.ch opcionais
  regras locais do Block Manager em faixa SID propria

12. GEOIP
---------

O motor foi compilado com libmaxminddb e GeoIP2 habilitado.

Modelo definido para o produto:
  Opcao 1: base nativa do Block Manager
  Opcao 2: MaxMind configurada pelo suporte/cliente

A politica do Block Manager deve armazenar paises por codigo ISO e ASN, sem acoplar regras ao fornecedor da base.

A base GeoIP nao faz parte deste pacote de motor 8.0.8_1. Ela sera gerenciada separadamente pelo Block Manager para poder ser atualizada sem recompilar o Suricata.

13. GRUPOS DO BLOCK MANAGER
---------------------------

O Suricata nao precisa conhecer LDAP, MAC, Captive ou usuarios pfSense diretamente.

O modelo ja validado do Block Manager transforma identidade em IPs e redes:

  identidade/dispositivo
    -> grupo efetivo
    -> IPs/CIDRs
    -> datasets/politica de seguranca
    -> Suricata/PF

Os datasets por grupo permitirao aplicar perfis diferentes de IDS, IPS, QUIC e GeoIP aproveitando o mesmo modelo de grupos usado pelos demais modulos.

14. EMPACOTAMENTO FREEBSD
-------------------------

Nome:
  blockmanager-suricata-engine

Versao:
  8.0.8_1

Origin:
  security/blockmanager-suricata-engine

Prefixo:
  /usr/local

Motor instalado em:
  /usr/local/blockmanager/suricata/bin/suricata
  /usr/local/blockmanager/suricata/bin/suricata-update
  /usr/local/blockmanager/suricata/bin/suricatactl
  /usr/local/blockmanager/suricata/bin/suricatasc

Configuracao-base:
  /usr/local/etc/blockmanager/suricata/

A WebGUI tradicional do pfSense Suricata NAO entra no pacote.
Foram validadas como ausentes entradas em:
  /usr/local/www/suricata*
  /usr/local/pkg/suricata*
  /usr/local/share/pfSense-pkg-suricata*

O pacote nao inicia o motor automaticamente e nao modifica PF.

15. DETALHE DO PKG-PLIST
------------------------

O +MANIFEST utiliza:
  prefix: /usr/local

Por isso, as entradas de pkg-plist sao relativas ao prefixo, por exemplo:
  blockmanager/suricata/bin/suricata
  etc/blockmanager/suricata/suricata.yaml

O root usado no pkg create e a raiz completa do payload:
  /root/blockmanager-build/suricata/pkgroot

Comando final:

  pkg create \
    -M /root/blockmanager-build/suricata/package/+MANIFEST \
    -p /root/blockmanager-build/suricata/package/pkg-plist \
    -r /root/blockmanager-build/suricata/pkgroot \
    -o /root/blockmanager-build/suricata/packages

Resultado:
  pkg_create_exit=0

Erro encontrado durante o processo:
  usar paths usr/local/... no plist junto com prefix /usr/local causava /usr/local/usr/local/...

Solucao:
  tornar as entradas do plist relativas a /usr/local e manter -r apontando para pkgroot.

16. PACOTE FINAL
----------------

Arquivo:
  blockmanager-suricata-engine-8.0.8_1.pkg

Tamanho compactado observado:
  aproximadamente 28 MB

Flat size:
  157 MiB

Arquitetura:
  FreeBSD:16:amd64

SHA-256:
  7a987e3ff7b3475cb6cbd2fbe4f6af60fc008791a10bb8be0cfed0b4554d278f

17. PROCEDIMENTO DE INSTALACAO DE LABORATORIO
---------------------------------------------

O pacote e seu arquivo .sha256 devem ser colocados em /tmp no pfSense 2.9 de laboratorio.

Antes de instalar:

  cd /tmp
  sha256 blockmanager-suricata-engine-8.0.8_1.pkg
  cat blockmanager-suricata-engine-8.0.8_1.pkg.sha256
  cat /etc/version
  freebsd-version -ku
  uname -K
  uname -m
  pkg config ABI
  pkg info -F /tmp/blockmanager-suricata-engine-8.0.8_1.pkg

Esperado:
  ABI FreeBSD:16:amd64

Depois deve ser feito preflight das dependencias no repositorio do pfSense antes de pkg add.

A instalacao inicial deve ser validada com:
  ldd /usr/local/blockmanager/suricata/bin/suricata
  /usr/local/blockmanager/suricata/bin/suricata --build-info
  suricata -T com configuracao do Block Manager

18. TESTE QUIC PLANEJADO
------------------------

A primeira execucao no pfSense deve ser IDS passivo, sem IPS e sem Netmap inline.

Objetivo:
  trafego UDP/443 real
    -> Suricata
    -> parser QUIC
    -> EVE JSON
    -> identificar origem/destino
    -> quic.sni quando disponivel
    -> quic.version
    -> regra local de teste

Somente depois da validacao IDS devem ser testados Netmap e IPS em interface/VLAN de laboratorio.

19. SEGURANCA E FAIL-SAFE
-------------------------

Principios definidos:
  - O pacote de motor nao altera regras PF.
  - O pacote de motor nao inicia automaticamente.
  - IPS fica desativado na primeira instalacao.
  - As configuracoes geradas devem passar por suricata -T antes da promocao.
  - Atualizacoes de regras trabalham em staging e so promovem configuracao valida.
  - Falha do Suricata nao deve liberar QUIC indiscriminadamente; PF permanece como enforcement/fail-safe do Block Manager.
  - Mudanca de datasets ou grupos nao deve provocar filter_configure completo do pfSense.

20. REPRODUCAO RESUMIDA
-----------------------

1. Provisionar FreeBSD 16 amd64 com recursos suficientes.
2. Instalar Rust, cbindgen, Python 3.12 e bibliotecas.
3. Baixar e registrar tarball Suricata 8.0.8.
4. Extrair em arvore limpa.
5. Configurar prefixo isolado e recursos Netmap/GeoIP.
6. Desabilitar march native.
7. Compilar com gmake -j1.
8. Executar install-full somente via DESTDIR.
9. Validar build-info, ldd e suricata -T.
10. Preparar pkgroot removendo caches gerados.
11. Criar +MANIFEST e pkg-plist relativo ao prefixo.
12. Executar pkg create.
13. Registrar SHA-256.
14. Instalar primeiro somente em pfSense de laboratorio.
15. Validar IDS/QUIC antes de IPS.

21. ARTEFATOS A PRESERVAR
-------------------------

Na maquina de build, preservar:
  source/suricata-8.0.8.tar.gz
  source/suricata-8.0.8.tar.gz.sig
  logs/suricata-8.0.8-source.sha256
  logs/configure-r3.log
  logs/build.log
  logs/build.exit
  logs/install-stage.log
  logs/build-info.txt
  logs/ldd.txt
  package/+MANIFEST
  package/pkg-plist
  packages/blockmanager-suricata-engine-8.0.8_1.pkg
  packages/blockmanager-suricata-engine-8.0.8_1.pkg.sha256

A VM de build pode permanecer desligada, mas nao deve ser descartada, pois e a referencia reprodutivel FreeBSD 16 para futuras recompilacoes e atualizacoes de seguranca.

22. PROXIMAS ETAPAS
-------------------

- Preflight das dependencias no pfSense 2.9.
- Instalar o pacote de laboratorio a partir de /tmp.
- Validar ldd, build-info e configuracao.
- Executar primeiro teste QUIC real em IDS.
- Validar quic.sni e EVE JSON.
- Criar controlador/servico Block Manager.
- Integrar grupos via datasets.
- Adicionar base GeoIP nativa e opcao MaxMind.
- Integrar ET Open/suricata-update.
- Homologar Netmap/IPS em VLAN de teste.
- Repetir build por ABI para plataformas pfSense/FreeBSD adicionais.

FIM
