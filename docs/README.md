# Curso de P4 com Mininet e BMv2

Este repositório contém materiais para um curso completo sobre P4 (Programming Protocol-independent Packet Processors), utilizando Mininet para emulação de rede e BMv2 (Behavioral Model version 2) como switch programável. O curso abrange desde os conceitos básicos da linguagem P4 até tópicos avançados e aplicações práticas.

## Conteúdo do Curso

### Slides da Apresentação

Os slides do curso estão disponíveis em formato HTML e cobrem os seguintes tópicos:

- Introdução ao P4
- Arquitetura P4
- Fundamentos da Linguagem P4
- BMv2 e Mininet
- Ambiente de Desenvolvimento
- Programação Básica em P4
- Recursos Intermediários do P4
- Programação Avançada em P4
- Casos de Uso e Aplicações
- Projeto Prático

---

## Guia Passo a Passo: Usando P4 com Mininet e BMv2

Este guia fornece instruções detalhadas para configurar e usar a linguagem P4 com Mininet e o Behavioral Model versão 2 (BMv2), permitindo a criação e teste de programas P4 em um ambiente virtualizado.

### Índice

1. [Instalação do Ambiente](#1-instalação-do-ambiente)
2. [Estrutura de um Programa P4](#2-estrutura-de-um-programa-p4)
3. [Compilação de Programas P4](#3-compilação-de-programas-p4)
4. [Criação de Topologias Mininet com BMv2](#4-criação-de-topologias-mininet-com-bmv2)
5. [Execução e Teste](#5-execução-e-teste)
6. [Depuração](#6-depuração)
7. [Exemplos Práticos](#7-exemplos-práticos)
8. [Solução de Problemas Comuns](#8-solução-de-problemas-comuns)

### 1. Instalação do Ambiente

#### 1.1. Requisitos do Sistema

- Sistema operacional Linux (Ubuntu 18.04 ou superior recomendado)
- Pelo menos 2 GB de RAM
- Pelo menos 20 GB de espaço em disco
- Acesso de administrador (sudo)

#### 1.2. Instalação Automatizada

A maneira mais simples de instalar o ambiente P4 completo é usando o script de instalação fornecido pelo repositório de tutoriais P4:

```bash
# Clone o repositório de tutoriais P4
git clone https://github.com/p4lang/tutorials

# Entre no diretório de tutoriais
cd tutorials

# Execute o script de instalação (pode levar algum tempo)
./vm-ubuntu-20.04/install-p4dev-v4.sh
```

Este script instalará:
- Compilador P4 (p4c)
- Behavioral Model v2 (BMv2)
- Mininet
- P4Runtime
- Ferramentas de depuração e análise

#### 1.3. Instalação Manual

Se preferir instalar os componentes manualmente:

##### 1.3.1. Dependências

```bash
sudo apt update
sudo apt install -y cmake g++ git automake libtool libgc-dev bison flex libfl-dev libgmp-dev libboost-dev libboost-iostreams-dev libboost-graph-dev llvm pkg-config python3 python3-pip python3-setuptools libprotobuf-dev protobuf-compiler libgrpc++-dev libgrpc-dev grpc-compiler libgtest-dev
```

##### 1.3.2. Compilador P4 (p4c)

```bash
git clone --recursive https://github.com/p4lang/p4c.git
cd p4c
mkdir build
cd build
cmake ..
make -j$(nproc)
sudo make install
cd ../..
```

##### 1.3.3. BMv2

```bash
git clone https://github.com/p4lang/behavioral-model.git
cd behavioral-model
./install_deps.sh
./autogen.sh
./configure --enable-debugger
make -j$(nproc)
sudo make install
cd ..
```

##### 1.3.4. Mininet

```bash
git clone https://github.com/mininet/mininet.git
cd mininet
util/install.sh -a
cd ..
```

### 2. Estrutura de um Programa P4

Um programa P4 básico consiste nos seguintes componentes:

#### 2.1. Cabeçalhos

Definição dos formatos de cabeçalho dos pacotes:

```p4
header ethernet_t {
    bit<48> dstAddr;
    bit<48> srcAddr;
    bit<16> etherType;
}

header ipv4_t {
    bit<4>  version;
    bit<4>  ihl;
    bit<8>  diffserv;
    bit<16> totalLen;
    bit<16> identification;
    bit<3>  flags;
    bit<13> fragOffset;
    bit<8>  ttl;
    bit<8>  protocol;
    bit<16> hdrChecksum;
    bit<32> srcAddr;
    bit<32> dstAddr;
}
```

#### 2.2. Parser

Define como extrair campos de cabeçalho dos pacotes:

```p4
parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {
    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            TYPE_IPV4: parse_ipv4;
            default: accept;
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
}
```

#### 2.3. Tabelas e Ações

Definem o processamento dos pacotes:

```p4
action drop() {
    mark_to_drop(standard_metadata);
}

action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
    standard_metadata.egress_spec = port;
    hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
    hdr.ethernet.dstAddr = dstAddr;
    hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
}

table ipv4_lpm {
    key = {
        hdr.ipv4.dstAddr: lpm;
    }
    actions = {
        ipv4_forward;
        drop;
        NoAction;
    }
    size = 1024;
    default_action = drop();
}
```

#### 2.4. Controle

Define a lógica de aplicação das tabelas:

```p4
control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {
    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
    }
}
```

#### 2.5. Deparser

Define como recompor os pacotes após o processamento:

```p4
control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.ipv4);
    }
}
```

### 3. Compilação de Programas P4

#### 3.1. Compilação para BMv2

Para compilar um programa P4 para o BMv2:

```bash
p4c-bm2-ss --p4v 16 -o <programa_compilado>.json <programa_fonte>.p4
```

Por exemplo:

```bash
p4c-bm2-ss --p4v 16 -o basic.json basic.p4
```

#### 3.2. Verificação de Erros

O compilador P4 fornecerá mensagens de erro detalhadas em caso de problemas no código. Certifique-se de resolver todos os erros antes de prosseguir.

### 4. Criação de Topologias Mininet com BMv2

#### 4.1. Script Python para Topologia

Crie um script Python para definir sua topologia Mininet com switches BMv2:

```python
#!/usr/bin/env python3

from mininet.net import Mininet
from mininet.topo import Topo
from mininet.log import setLogLevel, info
from mininet.cli import CLI
from mininet.link import TCLink

# Importe a classe P4Switch e P4Host
from p4_mininet import P4Switch, P4Host

import argparse
import os

# Defina sua topologia
class MyTopo(Topo):
    def __init__(self, sw_path, json_path, **opts):
        Topo.__init__(self, **opts)
        
        # Adicione hosts
        h1 = self.addHost('h1', ip="10.0.1.1/24", mac="00:00:00:00:01:01")
        h2 = self.addHost('h2', ip="10.0.2.2/24", mac="00:00:00:00:02:02")
        
        # Adicione switches P4
        s1 = self.addSwitch('s1',
                           sw_path=sw_path,
                           json_path=json_path,
                           thrift_port=9090)
        
        # Adicione links
        self.addLink(h1, s1)
        self.addLink(h2, s1)

def main():
    parser = argparse.ArgumentParser(description='Mininet topology script for P4 BMv2 switches')
    parser.add_argument('--behavioral-exe', help='Path to behavioral executable',
                        type=str, action='store', required=True)
    parser.add_argument('--json', help='Path to JSON config file',
                        type=str, action='store', required=True)
    args = parser.parse_args()
    
    # Crie a topologia
    topo = MyTopo(args.behavioral_exe, args.json)
    
    # Crie a rede
    net = Mininet(topo=topo,
                 host=P4Host,
                 switch=P4Switch,
                 controller=None)
    
    # Inicie a rede
    net.start()
    
    # Adicione regras de encaminhamento
    s1 = net.get('s1')
    s1.cmd('simple_switch_CLI --thrift-port 9090 < commands.txt')
    
    # Inicie o CLI do Mininet
    CLI(net)
    
    # Pare a rede quando terminar
    net.stop()

if __name__ == '__main__':
    setLogLevel('info')
    main()
```

#### 4.2. Arquivo de Comandos

Crie um arquivo `commands.txt` com comandos para configurar as tabelas do switch:

```
table_add ipv4_lpm ipv4_forward 10.0.1.1/32 => 00:00:00:00:01:01 1
table_add ipv4_lpm ipv4_forward 10.0.2.2/32 => 00:00:00:00:02:02 2
```

### 5. Execução e Teste

#### 5.1. Executando a Topologia

Execute sua topologia com o programa P4 compilado:

```bash
sudo python3 topology.py --behavioral-exe simple_switch --json basic.json
```

#### 5.2. Testando a Conectividade

No CLI do Mininet, teste a conectividade entre os hosts:

```
mininet> h1 ping h2
```

#### 5.3. Verificando Tabelas

Para verificar as tabelas configuradas no switch:

```
mininet> sh simple_switch_CLI --thrift-port 9090 -c "table_dump ipv4_lpm"
```

### 6. Depuração

#### 6.1. Logs do BMv2

Para habilitar logs detalhados no BMv2, adicione a opção `--log-console` ao executar o script de topologia:

```bash
sudo python3 topology.py --behavioral-exe "simple_switch --log-console" --json basic.json
```

#### 6.2. Usando o Debugger

O BMv2 inclui um debugger que permite inspecionar o estado interno do switch:

```bash
simple_switch_CLI --thrift-port 9090
```

Comandos úteis do debugger:
- `table_dump <table_name>`: Mostra o conteúdo de uma tabela
- `counter_read <counter_name> <index>`: Lê um contador
- `register_read <register_name> <index>`: Lê um registro
- `packet_in <hex_packet>`: Injeta um pacote no switch

### 7. Exemplos Práticos

#### 7.1. Exemplo de Encaminhamento Básico

Veja o exemplo completo de encaminhamento IPv4 básico em: https://github.com/p4lang/tutorials/tree/master/exercises/basic

#### 7.2. Exemplo de Tunneling

Veja o exemplo de implementação de tunneling em: https://github.com/p4lang/tutorials/tree/master/exercises/basic_tunnel

#### 7.3. Exemplo de Balanceamento de Carga

Veja o exemplo de balanceamento de carga em: https://github.com/p4lang/tutorials/tree/master/exercises/load_balance

### 8. Solução de Problemas Comuns

#### 8.1. Erro "No such file or directory"

**Problema**: Ao executar o script de topologia, você recebe um erro "No such file or directory".

**Solução**: Verifique se os caminhos para o executável BMv2 e o arquivo JSON estão corretos. O executável `simple_switch` deve estar no PATH do sistema.

#### 8.2. Erro "Thrift: TSocket read 0 bytes"

**Problema**: Ao tentar usar o CLI do simple_switch, você recebe um erro relacionado ao Thrift.

**Solução**: Verifique se o switch está em execução e se a porta Thrift especificada está correta.

#### 8.3. Pacotes não são encaminhados corretamente

**Problema**: Os hosts não conseguem se comunicar, mesmo com as regras de encaminhamento configuradas.

**Solução**:
1. Verifique se os cabeçalhos estão sendo analisados corretamente no parser.
2. Verifique se as regras de encaminhamento foram adicionadas corretamente.
3. Verifique se o TTL está sendo decrementado e se os pacotes não estão sendo descartados por TTL zero.
4. Use o comando `table_dump` para verificar se as regras estão instaladas corretamente.

#### 8.4. Problemas de Permissão

**Problema**: Erros de permissão ao executar scripts ou comandos.

**Solução**: Execute os comandos com `sudo` ou adicione seu usuário ao grupo apropriado.

### Conclusão

Este guia forneceu os passos básicos para começar a trabalhar com P4, Mininet e BMv2. Para aprender mais, consulte a documentação oficial e os tutoriais disponíveis em:

- [P4.org](https://p4.org/)
- [Repositório de Tutoriais P4](https://github.com/p4lang/tutorials)
- [Documentação do BMv2](https://github.com/p4lang/behavioral-model)
- [Documentação do Mininet](http://mininet.org/documentation/)

Lembre-se de que a programação P4 é uma habilidade que se desenvolve com a prática. Comece com exemplos simples e vá aumentando a complexidade à medida que se sentir mais confortável com a linguagem e as ferramentas.

---

## Script de Apresentação: Curso de P4 com Mininet e BMv2

Olá a todos! Sejam bem-vindos ao nosso curso de P4: Do Zero ao Avançado com Mininet e BMv2. Meu nome é [Seu Nome/Nome do Apresentador] e estou muito feliz em guiá-los por esta jornada no mundo da programação de redes.

---

### Slide 1: Introdução ao P4

**Título:** Introdução ao P4

**Pontos-chave:**
- O que é P4 (Programming Protocol-independent Packet Processors).
- Por que o P4 é importante: a evolução da programação de redes.
- Principais características: independência de protocolo, programabilidade do plano de dados, flexibilidade.

**Sugestão de Fala:**
"Começamos com a pergunta fundamental: o que é P4? P4, ou Programming Protocol-independent Packet Processors, é uma linguagem de domínio específico que nos permite descrever como os dispositivos de rede devem processar pacotes. Pensem em switches, roteadores, NICs – todos esses dispositivos podem ter seu comportamento de plano de dados programado com P4. Isso é uma mudança de paradigma enorme! Antes, os fabricantes ditavam o que um switch podia fazer. Com P4, nós, como engenheiros e desenvolvedores, ganhamos o poder de definir esse comportamento, abrindo portas para inovação e personalização sem precedentes. As principais características do P4 são sua independência de protocolo, a capacidade de programar diretamente o plano de dados e a flexibilidade que isso nos dá para inovar rapidamente."

---

### Slide 2: Arquitetura P4

**Título:** Arquitetura P4

**Pontos-chave:**
- Visão geral da arquitetura P4: Parser, Match-Action Pipeline, Deparser.
- Como os pacotes fluem através desses componentes.
- A importância de cada etapa no processamento de pacotes.

**Sugestão de Fala:**
"Para entender como o P4 funciona, precisamos olhar para sua arquitetura. Ela é composta por três blocos principais: o Parser, o Match-Action Pipeline e o Deparser. O Parser é o primeiro a agir, ele extrai as informações dos cabeçalhos dos pacotes. Em seguida, o pacote entra no coração do P4, o Match-Action Pipeline, onde as decisões são tomadas com base nas regras que programamos. Finalmente, o Deparser reconstrói o pacote, talvez com novos cabeçalhos ou modificações, antes de enviá-lo para fora do switch. Essa sequência de etapas nos dá um controle granular sobre cada pacote que passa pela rede."

---

### Slide 3: Fundamentos da Linguagem P4

**Título:** Fundamentos da Linguagem P4

**Pontos-chave:**
- Sintaxe básica: tipos de dados (bit<n>, int<n>, varbit<n>).
- Definição de cabeçalhos (header).
- Conceito de tabelas (table) e ações (action).

**Sugestão de Fala:**
"Agora, vamos mergulhar nos fundamentos da linguagem P4. A sintaxe é bastante intuitiva. Trabalhamos com tipos de dados como `bit<n>` para campos de tamanho fixo, `int<n>` para inteiros e `varbit<n>` para campos de tamanho variável. Um dos blocos de construção mais importantes são os cabeçalhos, onde definimos a estrutura dos protocolos que queremos processar, como Ethernet ou IPv4. E, claro, as tabelas e ações. As tabelas são onde definimos as regras de correspondência para os pacotes, e as ações são o que fazemos quando um pacote corresponde a uma dessas regras. É a combinação desses elementos que nos permite construir lógicas de encaminhamento complexas."

---

### Slide 4: BMv2 e Mininet

**Título:** BMv2 e Mininet

**Pontos-chave:**
- BMv2 (Behavioral Model version 2): o que é e sua função como switch P4 em software.
- Mininet: emulador de rede para criação de topologias virtuais.
- Como BMv2 e Mininet se integram para criar um ambiente de teste.

**Sugestão de Fala:**
"Para testar nossos programas P4, precisamos de um ambiente. É aí que entram o BMv2 e o Mininet. O BMv2, ou Behavioral Model version 2, é uma implementação de referência de um switch P4 em software. Ele nos permite executar e depurar nossos programas P4 sem a necessidade de hardware físico. O Mininet, por sua vez, é um emulador de rede que nos permite criar topologias de rede virtuais complexas em um único computador. A integração de BMv2 com Mininet é poderosa: podemos criar redes virtuais com switches P4 programáveis, testar nossos programas P4 e observar seu comportamento em um ambiente controlado e escalável."

---

### Slide 5: Ambiente de Desenvolvimento

**Título:** Ambiente de Desenvolvimento

**Pontos-chave:**
- Ferramentas necessárias: compilador P4 (p4c), BMv2, Mininet, P4Runtime.
- Processo de compilação de um programa P4 para BMv2.
- Execução de topologias Mininet com switches P4.

**Sugestão de Fala:**
"Configurar o ambiente de desenvolvimento é o primeiro passo prático. Precisamos do compilador P4, o `p4c`, que transforma nosso código P4 em um formato que o BMv2 entende. Também instalamos o próprio BMv2, o Mininet para a emulação de rede e o P4Runtime para a comunicação entre o plano de controle e o plano de dados. O processo é simples: escrevemos nosso código P4, compilamos com `p4c-bm2-ss`, e então usamos um script Python para criar nossa topologia Mininet, especificando que os switches devem usar o BMv2 com nosso programa P4 compilado. O guia passo a passo que disponibilizei detalha cada etapa da instalação e configuração."

---

### Slide 6: Programação Básica em P4

**Título:** Programação Básica em P4

**Pontos-chave:**
- Exemplo de implementação de uma ação de encaminhamento IPv4.
- Definição de uma tabela de encaminhamento (LPM).
- Como essas peças se encaixam para processar pacotes.

**Sugestão de Fala:**
"Vamos ver um exemplo prático de programação básica. Aqui temos uma ação simples, `ipv4_forward`, que modifica o cabeçalho Ethernet, decrementa o TTL do IPv4 e define a porta de saída. Essa ação é então associada a uma tabela, `ipv4_lpm`, que usa o endereço IP de destino para decidir qual ação tomar: encaminhar o pacote usando `ipv4_forward` ou descartá-lo. Este é o bloco fundamental para construir qualquer funcionalidade de encaminhamento em P4. O código completo está disponível nos tutoriais do P4 e no guia que forneci."

---

### Slide 7: Recursos Intermediários do P4

**Título:** Recursos Intermediários do P4

**Pontos-chave:**
- Tunneling e encapsulamento (VXLAN, GRE).
- Implementação de multicast.
- Controle de fluxo e Qualidade de Serviço (QoS).

**Sugestão de Fala:**
"Com os fundamentos em mente, podemos explorar recursos intermediários. O P4 nos permite implementar facilmente tunneling e encapsulamento, como VXLAN, para criar redes virtuais sobre uma infraestrutura física. Também podemos configurar multicast, enviando pacotes para múltiplos destinos de forma eficiente. E, crucialmente, o P4 nos dá o poder de implementar mecanismos sofisticados de controle de fluxo e Qualidade de Serviço (QoS) diretamente no plano de dados, garantindo que o tráfego crítico receba o tratamento adequado."

---

### Slide 8: Programação Avançada em P4

**Título:** Programação Avançada em P4

**Pontos-chave:**
- Telemetria In-band Network (INT).
- Balanceamento de carga avançado.
- Implementação de segurança e firewall no plano de dados.
- P4Runtime para interação com o plano de controle.

**Sugestão de Fala:**
"No nível avançado, o P4 brilha em áreas como a Telemetria In-band Network, ou INT, que nos permite coletar dados de telemetria detalhados diretamente do plano de dados, oferecendo visibilidade sem precedentes sobre o desempenho da rede. Podemos implementar algoritmos de balanceamento de carga complexos e construir firewalls e mecanismos de segurança de alto desempenho diretamente nos switches. E tudo isso é orquestrado pelo P4Runtime, uma API que permite que o plano de controle interaja dinamicamente com o plano de dados programado em P4."

---

### Slide 9: Casos de Uso e Aplicações

**Título:** Casos de Uso e Aplicações

**Pontos-chave:**
- Monitoramento de rede em tempo real.
- Balanceamento de carga inteligente.
- Segurança e mitigação de ataques.
- Virtualização de rede e novos protocolos.

**Sugestão de Fala:**
"Os casos de uso do P4 são vastos e impactantes. Desde o monitoramento de rede em tempo real, que nos dá insights instantâneos sobre o que está acontecendo, até o balanceamento de carga inteligente que otimiza a utilização de recursos. Na segurança, o P4 permite respostas rápidas a ameaças, implementando regras de firewall diretamente no hardware. E na virtualização de rede, ele nos dá a flexibilidade para criar e gerenciar redes virtuais com controle granular. O P4 está transformando a forma como construímos e gerenciamos redes."

---

### Slide 10: Projeto Prático

**Título:** Projeto Prático

**Pontos-chave:**
- Proposta de projeto: Implementar um sistema de balanceamento de carga.
- Etapas do projeto: topologia, encaminhamento básico, balanceamento, monitoramento.
- Aprendizado esperado: aplicação prática dos conceitos de P4, Mininet e BMv2.

**Sugestão de Fala:**
"Para consolidar o aprendizado, proponho um projeto prático: implementar um sistema de balanceamento de carga usando P4, Mininet e switches BMv2. Este projeto guiará vocês pelas etapas de criação de uma topologia, implementação de encaminhamento básico, adição da lógica de balanceamento de carga e, finalmente, o monitoramento do sistema. É uma excelente oportunidade para aplicar tudo o que vimos e ganhar experiência prática com a programação de redes programáveis."