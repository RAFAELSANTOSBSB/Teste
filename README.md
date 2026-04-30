# Contexto do Desafio Técnico

O sistema precisa ingerir dados de ECG em tempo real (~500 amostras/s por dispositivo), operando em ambientes com conectividade instável, garantindo integridade dos dados (LGPD) e permitindo evolução futura para processamento por machine learning sem reestruturação.

---

## 1. Buffer local no Edge vs Envio Direto ao Cloud

**Decisão:**
* Utilizar um **Edge Agent** com buffer local persistente (**SQLite**) para armazenar dados durante falhas de conectividade.

### Alternativas Consideradas:
* Envio direto para o cloud sem buffer.
* Uso de broker local na clínica.
* Armazenar em arquivo.

### Trade-offs:
* **Prós:** Garante integridade dos dados mesmo com quedas de até 30 minutos e reduz dependência da conectividade.
* **Contras:** Aumenta complexidade no Edge (persistência e retry).

> **Justificativa:** Dado o requisito de conectividade instável (R2), a perda de dados é inaceitável. O buffer local garante resiliência sem depender da infraestrutura da clínica. O sistema priorizaria dados em tempo real sobre backlog, garantindo latência para o dashboard enquanto drena dados históricos de forma controlada.

---

## 2. Uso combinado de MQTT + Kafka vs Uso de apenas um

**Decisão:**
* Utilizar **MQTT** para ingestão e **Kafka** como backbone de processamento.
* O broker MQTT é dimensionado para throughput, não retenção.

### Alternativas Consideradas:
* Apenas Kafka.
* Apenas MQTT.

### Trade-offs:
* **Prós:** MQTT é leve e adequado para edge e redes instáveis; Kafka fornece retenção, reprocessamento e escala.
* **Contras:** Introduz componente adicional (bridge).

> **Justificativa:** MQTT resolve ingestão em ambientes instáveis, enquanto Kafka atende requisitos de escala, reprocessamento e evolução (R1 e R4). A separação reduz acoplamento e melhora resiliência.

---

## 3. Uso de banco time-series vs Banco relacional tradicional

**Decisão:**
* Utilizar banco de dados **time-series** (ex: TimescaleDB ou ClickHouse) para armazenamento histórico.

### Alternativas Consideradas:
* PostgreSQL tradicional (possibilidade de adicionar o TimescaleDB).
* Armazenamento em arquivos.

### Trade-offs:
* **Prós:** Melhor performance para dados temporais; suporte nativo a particionamento e retenção.
* **Contras:** Maior complexidade operacional.

> **Justificativa:** O volume de dados (~500k eventos/s) e necessidade de retenção de 1 ano exigem um modelo otimizado para séries temporais, reduzindo custo e aumentando eficiência.

---

## 4. Separação de processamento e ML

**Decisão:**
* ML consome dados diretamente do Kafka como consumidor independente.

> **Justificativa:** Permite evolução do sistema sem impacto na pipeline principal, atendendo o requisito R4.

---

## Respostas do Questionário Técnico

### 1) Docker
* **(a) Estratégia:** Usaria multi-stage build: primeira stage com Go (build), segunda com imagem mínima (ex: alpine ou distroless). Criaria usuário não-root e ajustaria permissões para acessar `/dev/ttyUSB0` (via group dialout). Evitaria incluir ferramentas desnecessárias na imagem final.
* **(b) COPY vs cp:** `COPY` ocorre no build da imagem e gera uma nova camada imutável. `cp` atua em tempo de execução no container/host e não impacta a imagem. `COPY` é versionado e reprodutível; `cp` é imperativo e não rastreável.
* **(c) Produção:** Acesso ao `/dev/ttyUSB0` pode falhar em produção por permissões ou falta de mapeamento do device no container (`--device`), mesmo funcionando localmente.

### 2) Mensageria
* **Escolha:** MQTT + Kafka.
* **Papéis:** MQTT na ingestão no edge (leve, tolerante a conexões instáveis, suporte a QoS/IoT); Kafka como backbone de streaming (retenção, reprocessamento e múltiplos consumidores).
* **Trade-offs:** Melhor desacoplamento; escalabilidade; maior complexidade operacional (bridge + dois sistemas). A separação evita sobrecarregar o MQTT com retenção e permite evolução futura.

### 3) Segurança
* **(a) OAuth2 vs OIDC:** OAuth2 é suficiente para autorização (delegação de acesso). OIDC é necessário para autenticação, pois adiciona identidade (ID Token) sobre OAuth2.
* **(b) Risco:** Usar OAuth2 como autenticação é arriscado porque não garante identidade do usuário, apenas autorização. Pode permitir acesso indevido se tokens forem mal interpretados, violando LGPD.

### 4) Banco de dados
* **Abordagem:** Banco time-series (TimescaleDB ou ClickHouse) para histórico (particionamento por tempo e TTL de 1 ano). Redis para o dashboard em tempo real (últimos 30s).
* **Trade-offs:** Alta performance para escrita/consultas temporais; redução de custo com retenção automática; maior complexidade operacional.

### 5) Visão sistêmica
* **Escolha:** Opção (B) - reutilizar Kafka existente como backbone, aproveitando infraestrutura e conhecimento disponíveis.
* **Estratégia:** Introduzir MQTT apenas na camada de ingestão (edge).
* **Impacto Mínimo:** Definir tópicos isolados (`ecg.raw`, `ecg.processed`), estabelecer quotas/limites, padronizar producers/consumers e alinhar governança.
* **Trade-offs:** Reuso de infraestrutura; MQTT resolve rede instável; aumento de complexidade na borda (bridge).
