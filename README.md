# 🕵️‍♀️ Análise de Tráfego de Rede com Wireshark
### ARP Spoofing & TCP SYN — Tráfego Real vs. Tráfego sob Ataque

![Visão geral](images/00-visao-geral-diagrama.jpg)

---

## 🎯 Objetivo

Capturar tráfego da minha própria rede doméstica e comparar com um pcap de laboratório contendo um ataque simulado de **ARP Spoofing**, com o objetivo de identificar, na prática, os sinais que diferenciam uma rede funcionando normalmente de uma rede comprometida — olhando tanto para o comportamento do protocolo **ARP** quanto para a estrutura dos pacotes **TCP (SYN)**.

---

## 🛠️ Ferramentas utilizadas

- **Wireshark**
- **Sistema operacional:** Windows 11 (64-bit) — captura própria, interface Ethernet

---

## 🧪 Metodologia

| | Captura própria | Pcap de laboratório |
|---|---|---|
| **Interface** | Ethernet (Npcap) | — (arquivo pronto, sem captura ao vivo) |
| **Filtro de captura** | Nenhum | — |
| **Duração** | ~83 segundos | ~209 segundos (simulado) |
| **Pacotes** | 1075 | 184 |

Em ambos os casos, a técnica de investigação foi a mesma: aplicar filtros de exibição sucessivos no Wireshark — `arp` e `tcp.flags.syn == 1` — e comparar o resultado entre as duas capturas.

---

## 📦 Pacotes analisados

### 🔗 ARP — cada IP deveria ter sempre o mesmo MAC

**Rede normal:** cada IP responde sempre com o mesmo MAC, do início ao fim da captura.

![ARP em rede normal](images/01-arp-rede-normal.jpg)

**Rede sob ataque:** a partir de ~61,5s, o MAC `aa:bb:cc:dd:ee:ff` começa a responder por **dois IPs diferentes** (`192.168.1.1` e `192.168.1.10`), sem ninguém ter perguntado — ARP gratuito.

![ARP sob ataque](images/02-arp-rede-ataque.jpg)

> ⚠️ **Não é uma alternância organizada.** A resposta legítima só aparece quando alguém pergunta de novo (raro, ~a cada 40s). A resposta falsa chega sem ninguém pedir, a cada ~10s, quase sem parar — o atacante inunda a rede muito mais rápido do que o gateway real consegue se corrigir.

### 🤝 TCP SYN — a estrutura do pacote também conta uma história

**Rede normal:** SYNs de 66–86 bytes, com opções TCP completas (`MSS`, `Window Scaling`, `SACK_PERM`) — negociações que um sistema operacional real sempre faz.

**Laboratório:** SYNs de exatamente 54 bytes — o mínimo possível, **sem nenhuma opção TCP.**

![Comparação de pacotes SYN](images/05-syn-comparacao.jpg)

![O que isso revela](images/06-syn-o-que-revela.jpg)

---

## 🔍 Observações e conclusões

✅ **O que se comportou como esperado:** o three-way handshake TCP (SYN → SYN-ACK → ACK) ocorreu corretamente nas duas capturas, sem anomalias na camada de transporte.

🚩 **O que foi inesperado:**
- Um único MAC (`aa:bb:cc:dd:ee:ff`) se anunciando como dono de dois IPs diferentes — assinatura clássica de **ARP Spoofing / Man-in-the-Middle**.
- SYNs sem nenhuma opção TCP no cenário de ataque, junto com um padrão de conexões novas e repetitivas a cada poucos segundos — indício de tráfego gerado por script, não por navegação humana.

💡 **Aprendizados:**
- Numa rede saudável, IP e MAC mantêm uma relação **estável**; um IP "trocando" de MAC é o sinal mais confiável de ARP Spoofing.
- A **estrutura** de um pacote (tamanho, opções TCP, frequência de conexões) pode denunciar automação, mesmo sem olhar o conteúdo.
- O próprio Wireshark ajuda na detecção automática: *Analyze > Expert Information* sinaliza o conflito como **"Duplicate IP address configured"**.

---

## 🔁 Como reproduzir

1. Abra o Wireshark e selecione a interface de rede ativa (Ethernet ou Wi-Fi).
2. Inicie a captura **sem filtro** (*Start Capturing*).
3. Gere tráfego de propósito: acesse um site no navegador e dê um ping no gateway (`ping <IP do gateway>`, obtido via `ipconfig` no Windows).
4. Após 1–2 minutos, pare a captura e salve como `.pcapng` (*File > Save As*).
5. Aplique os filtros de exibição, um de cada vez, e compare com os prints acima:
   - `arp` → confira se cada IP aponta sempre para o mesmo MAC.
   - `tcp.flags.syn == 1` → confira o tamanho dos pacotes e se aparecem opções TCP (clique em cada SYN e expanda "Transmission Control Protocol > Options").
6. Se quiser comparar com um cenário de ataque, use o pcap de laboratório (`lab_A08_ecommerce_incidentv2_pcap.pcap`) como referência.

---

*Projeto desenvolvido como parte do módulo de Cibersegurança 2.0 — Kodie Academy.*

