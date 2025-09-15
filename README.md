# Ataques DDoS — Explicação Formal, Detalhada e Completa

> **Resumo:** Oferecer uma visão técnica e didática sobre ataques DDoS (Distributed Denial of Service). Aborda definições, vetores de ataque, mecanismos comuns (botnets, spoofing, amplificação), impactos, métricas usadas para caracterização, técnicas de detecção e mitigação, resposta a incidentes, aspectos legais e referências com imagens públicas encontradas na internet.

---
## 1. Introdução

Um ataque de negação de serviço distribuído (DDoS) é um esforço malicioso para tornar um serviço (site, API, servidor, rede) indisponível para seus usuários legítimos, saturando sua capacidade de processamento, memória, recursos de conexão ou largura de banda. A distribuição (do termo "distributed") significa que o tráfego de ataque provém de múltiplas origens, o que torna a mitigação e a atribuição mais difíceis.

## 2. Como funciona um ataque DDoS

Em essência, um atacante reúne fontes capazes de gerar grande volume de solicitações (chamadas de *bots* ou *zombies*, coletivamente *botnet*) e as instrui a enviar tráfego ao alvo. O efeito pretendido pode ser:

* Saturar a **largura de banda** da conectividade do alvo (impedindo que tráfego legítimo chegue);
* Esgotar **recursos do servidor** (CPU, memória, conexões abertas, tabelas de estado em firewalls/load balancers);
* Degradar a **experiência do usuário** por aumento de latência, erros, timeouts.

Importante: ataques modernos frequentemente usam múltiplas técnicas ao mesmo tempo (*multi‑vector*), combinando volumétrico com ataques à camada de aplicação.

## 3. Classificação / Tipos de ataques

A classificação dos ataques DDoS ajuda a entender o objetivo do atacante e qual camada do modelo OSI está sendo explorada. A seguir são descritos os principais grupos com foco em comportamento, impacto e métricas de medição.

### 3.1 Ataques volumétricos

**Descrição:** visam consumir a capacidade de transmissão (largura de banda) entre o alvo e a Internet.

**Vetores típicos:** UDP flood, ICMP flood, tráfego gerado por ataques de reflexão/amplificação (DNS, NTP, Memcached, CLDAP).

**Como são medidos:** em **bps (bits por segundo)** — ex.: Mbps, Gbps, Tbps.

**Sintomas no alvo:** saturação do link (pacotes perdidos, latência elevada), falha na entrega de pacotes legítimos, degradação de serviços distribuídos.

**Dificuldade de mitigação:** requer capacidade de absorção e filtragem em pontos de borda (provedores, CDNs), pois o pico pode exceder recursos locais.

### 3.2 Ataques de protocolo (state‑exhaustion)

**Descrição:** procuram esgotar estruturas de estado em equipamentos de rede ou servidores (tabelas de conexões, estados de sessões, recursos kernel).

**Vetores típicos:** SYN flood (TCP handshake incompleto), ataques que abusam de parâmetros de protocolo (ex.: fragmentação maliciosa), conntrack exhaustion em firewalls.

**Como são medidos:** em **número de conexões por segundo**, taxa de novos handshakes, ou número de estados simultâneos consumidos.

**Sintomas no alvo:** novos clientes não conseguem estabelecer conexão, aumento de conexões incompletas, consumo de memória/CPU em appliances de borda.

**Dificuldade de mitigação:** pode ser tratada com hardening TCP (SYN cookies), aumento de capacidade de tabelas e políticas específicas em firewalls/load balancers.

### 3.3 Ataques à camada de aplicação (Layer 7)

**Descrição:** simulam comportamento legítimo ao nível da aplicação (HTTP, DNS, SMTP, APIs) para forçar processamento dispendioso no servidor alvo.

**Vetores típicos:** HTTP GET/POST floods, ataques que exploram endpoints caros (pesquisas com joins complexos, geração de relatórios), Slowloris (manter conexões abertas lentamente).

**Como são medidos:** **RPS (requests per second)**, latência por endpoint, taxa de erros 5xx.

**Sintomas no alvo:** alto uso de CPU, aumento nas latências das respostas, filas de requisições, degradação de banco de dados e serviços dependentes.

**Dificuldade de mitigação:** geralmente exige análise semântica do tráfego, WAF eficiente, rate‑limiting e segregação de endpoints críticos; ataques bem modelados podem imitar usuários legítimos e driblar regras simples.

### 3.4 Ataques híbridos / multi‑vector

**Descrição:** combinam estratégias acima (ex.: amplificação para saturar link + Layer 7 para derrubar aplicação). São cada vez mais comuns porque aumentam a eficácia e complicam a resposta.

**Observação prática:** a defesa precisa ser igualmente multi‑camadas: detecção de anomalias, absorção de tráfego em provedores/CDNs e regras aplicacionais dinâmicas.

## 4. Técnicas e vetores comuns

Abaixo descrevemos com maior detalhe as técnicas que atacantes usam como base para criar os vetores acima. As explicações são enfocadas no que é observado em incidentes reais e em como identificar/mitigar — não há instruções de como executar ataques.

### 4.1 Botnets

**O que são:** redes de dispositivos comprometidos (desktops, servidores, dispositivos IoT) controladas por um operador (C2 — command and control).

**Características:** grande escala (milhares a milhões de bots), diversidade geográfica e de capacidade, persistência (bots se reconectam ao C2).

**Sinais de uso:** tráfego coordenado de muitos IPs distintos, picos sincronizados, assinaturas semelhantes em User‑Agent ou padrões de requisição.

**Mitigação:** listas de bloqueio de IPs conhecidos, solução anti‑bot com desafio (CAPTCHA), integração com provedores de threat intelligence para bloquear ASNs maliciosos, segmentação e rate limiting por IP/usuário.

### 4.2 IP spoofing, reflexão e amplificação

**IP spoofing:** o atacante falsifica o endereço IP de origem nos pacotes. Isso dificulta atribuição e pode direcionar respostas a uma vítima inocente.

**Reflexão:** o atacante envia uma requisição a um serviço aberto (por exemplo, servidor DNS recursivo) usando o IP da vítima como origem; a resposta do servidor recai sobre a vítima.

**Amplificação:** quando a resposta do serviço é muito maior que a requisição, o atacante amplifica o volume de tráfego enviado ao alvo com pouco esforço (ex.: requisição pequena gerando resposta grande).

**Serviços comumente abusados:** DNS (respostas maiores), NTP (monlist em implementações antigas), Memcached (respostas massivas), CLDAP, SSDP.

**Sinais de ataque:** grandes volumes de tráfego de portas/serviços específicos (ex.: UDP/53 para DNS), padrões de payload repetitivo, aumento de respostas por segundo vindas de muitos servidores reflexivos.

**Mitigação:** fechamento de serviços abertos (hardening), aplicação de egress filtering (provedores bloqueando spoofed packets — RFC 2827/BCP 38), rate limiting em servidores que respondem externamente.

### 4.3 Evasão e randomização

**Técnica:** variar cabeçalhos HTTP, rotacionar User‑Agents, usar proxies ou redes de bots para mimetizar tráfego legítimo.

**Impacto na defesa:** regras estáticas baseadas em assinaturas tornam‑se ineficazes; exige análise comportamental e heurística (anomaly detection) e desafios adaptativos.

**Mitigação:** soluções de WAF com aprendizado de perfil, desafio JavaScript/CAPTCHA dinamicamente ativados, verificação de comportamento (tempo entre ações, padrões de navegação).

### 4.4 Ataques dirigidos a APIs e endpoints específicos

**Descrição:** atacantes investigam endpoints que exigem alto custo computacional (consultas caras, geração de PDFs, endpoints que disparam jobs) e os sobrecarregam.

**Sinais:** aumento desproporcional em requisições a poucos endpoints; uso de parâmetros que ampliam custo (largas ranges, filtros complexos).

**Mitigação:** proteções específicas por endpoint (rate limit, quotas de API keys, circuit breakers, caching agressivo para respostas caras).

### 4.5 Técnicas de persistência e acompanhamento

**Descrição:** mesmo após mitigação inicial, atacantes podem manter um nível baixo de tráfego (low‑and‑slow) para drenar recursos ou testar defesas.

**Detecção:** monitoramento contínuo, análise de tendências e alertas para pequenos desvios que persistem no tempo.

**Mitigação:** playbooks que incluem monitoramento pós‑incidente, ajustes de thresholds e bloqueio de padrões recorrentes.

---

(Se quiser, posso agora: 1) inserir imagens ilustrativas nessas seções com créditos; 2) adicionar exemplos de logs/alertas típicos para cada vetor; 3) transformar estas seções em um slide ou PDF.)

* **Throughput / largura de banda (Gbps/Tbps)** — quão grande é o volume de dados.
* **RPS (requests per second)** — quantas requisições por segundo são enviadas (útil para Layer 7).
* **Conexões concorrentes / taxa de novas conexões por segundo** — afeta servidores que mantêm estados.
* **Tempo médio de resposta / Errors per second** — afetam experiência do usuário.

Impactos: interrupção de serviços, perda de receita, custo operacional (mobilização de equipe, uso de mitigação em nuvem), danos à reputação e possíveis consequências legais dependendo do contexto.

## 6. Detecção e monitoramento

Sinais típicos de um DDoS em andamento:

* Surtos súbitos e não previstos no tráfego (pico de bytes/pps);
* Aumento de requisições por endpoint específico (URLs, APIs);
* Aumento de erros (5xx) e timeouts dos serviços;
* Padrões incomuns: IPs geograficamente dispersos, User‑Agents repetitivos, ou, ao contrário, alta randomização dependendo do ataque.

Ferramentas de monitoramento: sistemas de métricas (Prometheus, Grafana), IDS/IPS, NetFlow/sFlow, e soluções específicas de mitigação com dashboards de tráfego.

## 7. Mitigação e proteções práticas (defesa)

**Importante:** a seção abaixo descreve estratégias defensivas e boas práticas. Não contém instruções para executar ataques.

### Estratégias gerais

* **Provisionamento e redundância:** arquitetar serviços com capacidade de escala horizontal, balanceadores geográficos e CDNs para absorver tráfego legítimo e distribuir carga.
* **CDN / Scrubbing / DDoS protection as a service:** provedores especializados (Cloudflare, Akamai, Radware, etc.) filtram tráfego e absorvem picos volumétricos.
* **Rate limiting e políticas na camada de aplicação:** limitar requisições por IP/usuário e aplicar quotas para endpoints sensíveis.
* **Filtragem de tráfego (ACLs) e regras de rede:** bloquear padrões conhecidos, aplicar listas de bloqueio e regras em firewalls/routers.
* **WAF (Web Application Firewall):** defender contra comportamento malicioso na camada 7.
* **Anycast e roteamento inteligente:** distribuir o tráfego para múltiplos pontos POP para reduzir impacto em um único data center.
* **Proteções anti‑amplificação:** cerrar serviços abertos (ex.: servidores DNS recursivos abertos) e aplicar políticas de rate limiting em servidores que possam ser abusados.

### Controles técnicos adicionais

* **SYN cookies / TCP hardening** para mitigar SYN floods;
* **Anomaly detection com ML** para identificar padrões fora do normal (com cuidado para evitar falsos positivos);
* **Blackholing / sinkholing** como medida de último recurso (descarta o tráfego — também bloqueia usuários legítimos);
* **Playbooks automatizados** para escalonamento com provedores e times de rede/segurança.

## 8. Resposta a incidentes (orientações)

1. **Identificação:** confirmar que o evento não é um pico legítimo (lançamento, campanha de marketing).
2. **Escopo:** coletar métricas (RPS, Gbps, endpoints afetados, ASN de origem) e rotas envolvidas.
3. **Mitigação imediata:** ativar regras de rate limit, WAF, ou redirecionar tráfego para scrubbing service.
4. **Comunicação:** informar partes interessadas internas e, se aplicável, clientes; manter logs preservados para investigação.
5. **Remediação e pós‑análise:** revisar lacunas, ajustar proteções e atualizar playbooks.

## 9. Aspectos legais e éticos

Lançar um ataque DDoS é crime na maioria das jurisdições — tratar qualquer investigação com equipes legais e forenses. Organizações que investigam incidentes devem atuar respeitando leis locais e acordos de privacidade.

## 10. Boas práticas e recomendações para organizações

* Testar planos de resposta com exercícios de tabletop (não testes ativos de ataque sem autorização explícita).
* Manter inventário de pontos de exposição (DNS, serviços públicos, APIs).
* Configurar alertas baseados em anomalias de tráfego e health checks.
* Contratar proteção DDoS de provedores especializados quando apropriado.
* Educar times técnicos sobre diferenciação entre picos legítimos e ataques.



**Aviso ético e legal:** Este documento tem finalidade **educacional** e de defesa. A divulgação de técnicas de ataque com instruções passo‑a‑passo não é suportada aqui. Realizar ataques DDoS é ilegal na maioria dos países e pode acarretar responsabilidade criminal.

