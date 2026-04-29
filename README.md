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