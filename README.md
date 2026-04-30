Contexto do desavio tecnico
O sistema precisa ingerir dados de ECG em tempo real (~500 amostras/s por dispositivo), operando em ambientes com conectividade instável, 
garantindo integridade dos dados (LGPD) e permitindo evolução futura para processamento por machine learning sem reestruturação.

1. Buffer local no Edge vs envio direto ao cloud
Decisão: 
	- Utilizar um Edge Agent com buffer local persistente (SQLite) para armazenar dados durante falhas de conectividade.

Alternativas consideradas:
	- Envio direto para o cloud sem buffer
	- Uso de broker local na clínica
	- Armazenar em file.
Trade-offs
	- Garante integridade dos dados mesmo com quedas de até 30 minutos
	- Reduz dependência da conectividade
	- Aumenta complexidade no Edge (persistência e retry)
Justificativa:
	- Dado o requisito de conectividade instável (R2), a perda de dados é inaceitável. O buffer local garante resiliência sem depender 
	da infraestrutura da clínica.
	- O sistema priorizaria dados em tempo real sobre backlog, garantindo latência para o dashboard enquanto drena dados históricos de forma controlada.

2. Uso combinado de MQTT + Kafka vs uso de apenas um
Decisão:
	- Utilizar MQTT para ingestão e Kafka como backbone de processamento
	- O broker MQTT é dimensionado para throughput, não retenção.

Alternativas consideradas:
	- Apenas Kafka
	- Apenas MQTT
Trade-offs
	- MQTT é leve e adequado para edge e redes instáveis
	- Kafka fornece retenção, reprocessamento e escala
	- Introduz componente adicional (bridge)
Justificativa:
	- MQTT resolve ingestão em ambientes instáveis, enquanto Kafka atende requisitos de escala, reprocessamento e evolução (R1 e R4). 
	A separação reduz acoplamento e melhora resiliência.

3. Uso de banco time-series vs banco relacional tradicional
Decisão:
	- Utilizar banco de dados time-series (ex: TimescaleDB ou ClickHouse) para armazenamento histórico

Alternativas consideradas
	- PostgreSQL tradicional, da para adicionar o TimescaleDB
	- Armazenamento em arquivos
Trade-offs
	- Melhor performance para dados temporais
	- Suporte nativo a particionamento e retenção
	- Maior complexidade operacional
Justificativa:
	- O volume de dados (~500k eventos/s) e necessidade de retenção de 1 ano exigem um modelo otimizado para séries temporais, reduzindo custo e aumentando eficiência.

4. Separação de processamento e ML
Decisão:
	- ML consome dados diretamente do Kafka como consumidor independente
Justificativa:
	- Permite evolução do sistema sem impacto na pipeline principal, atendendo o requisito R4.
	
	
1) Docker (6–8 linhas)
(a) Usaria multi-stage build: primeira stage com Go (build), segunda com imagem mínima (ex: alpine ou distroless). Criaria usuário não-root e ajustaria permissões para acessar /dev/ttyUSB0 (via group dialout). Evitaria incluir ferramentas desnecessárias na imagem final.
(b) COPY ocorre no build da imagem e gera uma nova camada imutável. cp atua em tempo de execução no container/host e não impacta a imagem. COPY é versionado e reprodutível; cp é imperativo e não rastreável.
(c) Acesso ao /dev/ttyUSB0 pode falhar em produção por permissões ou falta de mapeamento do device no container (--device), mesmo funcionando localmente.

2) Mensageria (8–10 linhas)
Escolho MQTT + Kafka. MQTT é usado na ingestão no edge por ser leve, tolerante a conexões instáveis e suportar QoS e feito para Iot, atendendo o cenário de clínicas com internet intermitente. 
Kafka é usado como backbone de streaming, permitindo retenção, reprocessamento e consumo por múltiplos serviços (ex: ML).
Trade-offs:
	- Melhor desacoplamento entre ingestão e processamento 
	- Escalabilidade e suporte a múltiplos consumidores 
	- Maior complexidade operacional (bridge + dois sistemas) 
A separação evita sobrecarregar o MQTT com retenção e permite evolução futura sem redesign.

3) Segurança (até 8 linhas)
(a) OAuth2 é suficiente para autorização (delegação de acesso a recursos). OIDC é necessário quando precisamos de autenticação, pois adiciona identidade (ID Token) sobre OAuth2.
(b) Usar OAuth2 como autenticação é arriscado porque ele não garante identidade do usuário, apenas autorização. Isso pode permitir acesso indevido se tokens forem mal interpretados, violando requisitos de segurança e LGPD.

4) Banco de dados (6–8 linhas)
Utilizaria um banco time-series (ex: TimescaleDB ou ClickHouse) para armazenamento histórico, com particionamento por tempo (ex: diário) e TTL de 1 ano. Para o dashboard em tempo real (últimos 30s), usaria Redis.
Trade-off:
	- Alta performance para escrita e consultas temporais 
	- Redução de custo com retenção automática 
	- Maior complexidade operacional comparado a banco relacional simples 
Essa abordagem separa acesso em tempo real de histórico, otimizando ambos os casos.

5) Visão sistêmica (10–12 linhas)
Escolheria o (B), reutilizar Kafka existente como backbone de streaming, aproveitando infraestrutura e conhecimento já disponíveis.
Para atender dispositivos em ambientes com conectividade instável, introduzo MQTT apenas na camada de ingestão (edge), sem substituir o Kafka.
Para minimizar impacto:
	- Definir tópicos isolados (ecg.raw, ecg.processed) 
	- Estabelecer quotas e limites de throughput 
	- Padronizar producers/consumers 
	- Alinhar governança com outros times 
Trade-off:
	- Reuso de infraestrutura existente 
	- MQTT resolve ingestão em rede instável 
	- Aumento de complexidade na borda (bridge) 
Essa abordagem mantém Kafka como padrão organizacional e adiciona MQTT apenas onde é necessário.
