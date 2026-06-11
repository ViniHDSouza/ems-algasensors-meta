# ems-algasensors-meta

Para abrir o visualizado do rabbitMQ  http://localhost:15672/
usuario e senha esta no docker-compose.yml
usuario: rabbitmq
senha: rabbitmq

1. Criar a Exchange

Na aba Exchanges → Add New Exchange, configurar:

Name:temperature-processing.temperature-received.v1.e
Type: Fanout
Durable:true (persiste no disco após reinício)

2. Passo 2 — Criar a Queue

Na aba Queues & Streams → Add New Queue, configurar:
Name: temperature-monitoring.process-temperature.v1.q
Durable:true (padrão)
Arguments: nenhum