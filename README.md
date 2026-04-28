## Arquitetura AWS — FarmaHub Distribuição

**Data:** 28 de abril de 2026

**Empresa:** FarmaHub Distribuição

**Responsável:** Gabriel Veras

---

## Introdução

Este relatório detalha a implementação de serviços AWS na FarmaHub Distribuição, empresa
farmacêutica B2B de porte intermediário-pequeno, atualmente sem infraestrutura cloud.
O foco é a redução de custos operacionais com migração faseada e de baixo risco.

---

## Etapa 1 — Amazon S3 (Armazenamento Inteligente)

**Foco:** Substituição de servidores físicos de armazenamento

**Descrição técnica:**

O Amazon S3 será o repositório central de dados da FarmaHub, substituindo os servidores
físicos que hoje armazenam registros de distribuição, inventário e pedidos. O S3 opera
no modelo de object storage: cada arquivo é armazenado como um objeto com metadados
associados, acessível via HTTP/S sem necessidade de sistema de arquivos tradicional.

Para o perfil da FarmaHub, a estratégia de classes de armazenamento será:

- **S3 Standard:** dados ativos dos últimos 90 dias — pedidos em andamento,
  inventário corrente e registros de lote recentes. Latência de milissegundos e
  alta disponibilidade (99,99%).

- **S3 Standard-Infrequent Access (S3 Standard-IA):** dados com acesso
  ocasional, entre 90 dias e 1 ano — histórico de pedidos concluídos, relatórios
  mensais e registros de rastreabilidade de lotes encerrados. Mesmo desempenho
  do Standard, com custo por GB ~60% menor, mas com cobrança por recuperação.

- **S3 Glacier Instant Retrieval:** arquivos com acesso raro, acima de 1 ano —
  backups regulatórios e documentação ANVISA que precisa estar disponível em
  milissegundos quando exigida em auditorias. Custo por GB ~80% menor que o
  Standard.

A transição entre classes será gerenciada automaticamente via **S3 Lifecycle Policies**,
sem intervenção manual.

**Prazo estimado:** 4 semanas
*(Semanas 1–2: inventário e mapeamento de dados legados; Semanas 3–4: migração,
configuração de lifecycle policies e testes de integridade)*

**Ganho estimado:** 30–35% de redução em custos de armazenamento

**Baseline e justificativa:** empresas deste porte tipicamente operam com servidores
físicos ou NAS com custo médio de R$ 0,08–0,12/GB/mês, considerando hardware,
energia, licença e manutenção. O S3 Standard-IA e Glacier reduzem esse custo para
R$ 0,02–0,04/GB/mês para a maior parte do volume histórico. O ganho de 30–35% é
conservador porque considera que uma parcela relevante dos dados permanecerá em
Standard (acesso frequente), sem benefício de tiering. 

---

## Etapa 2 — AWS Lambda + Amazon EventBridge + Amazon SQS

**Foco:** Automação serverless com resiliência a picos de volume

**Descrição técnica:**

O **AWS Lambda** executará funções disparadas por eventos para automação operacional:
alertas de validade de medicamentos, reconciliação de pedidos e notificações de
reposição. Lambda cobra apenas pelo tempo de execução (por 100ms) e pelo número de
invocações, sem custo em ociosidade — modelo adequado para cargas intermitentes como
as da FarmaHub.

O **Amazon EventBridge** atuará como barramento de eventos central, roteando eventos
de negócio (novo pedido registrado, lote próximo ao vencimento, divergência de
inventário) para as funções Lambda corretas. Regras baseadas em schedule (cron)
gerenciarão processos recorrentes, como reconciliações noturnas.

O **Amazon SQS** (Simple Queue Service) será introduzido como camada de buffer entre
o EventBridge e o Lambda para cenários de pico — como fechamento de pedidos em
grandes lotes ou processamento de notas fiscais em volume. Sem o SQS, picos de
eventos podem gerar throttling no Lambda ou perda de invocações. A fila SQS garante
que cada evento seja processado exatamente uma vez (modo FIFO), com retry automático
em caso de falha, sem custo adicional relevante para o volume esperado.

**Prazo estimado:** 4 semanas
*(Semana 1: mapeamento dos eventos de negócio e definição de payloads; Semana 2:
desenvolvimento e teste das funções Lambda; Semanas 3–4: integração com EventBridge,
configuração das filas SQS e testes de carga)*

**Ganho estimado:** 50–60% de redução em custos computacionais em relação ao modelo atual

**Baseline e justificativa:** o baseline considera servidores ligados continuamente
para executar rotinas que, na prática, rodam por minutos ao dia — perfil típico de
empresas deste porte. Um servidor EC2 t3.medium rodando 24/7 custa aproximadamente
USD 30/mês. O equivalente em Lambda para as mesmas rotinas, com execuções de 1–2
minutos distribuídas ao longo do dia, ficaria abaixo de USD 5/mês. 

---

## Etapa 3 — Amazon Redshift Serverless

**Foco:** Analytics operacional sobre dados logísticos

**Descrição técnica:**

O **Amazon Redshift Serverless** será o data warehouse analítico da FarmaHub, com
foco em análise de rotas de distribuição, sazonalidade de demanda e previsão de
necessidade por região. O modelo Serverless cobra por **RPU-segundo** (Redshift
Processing Unit), sem custo de infraestrutura parada — adequado para empresas deste
porte que não têm equipe de analytics rodando queries continuamente.

A ingestão de dados do S3 será feita via **COPY command** e complementada com
**Redshift Spectrum** para consultas híbridas — dados quentes no warehouse, dados
históricos frios acessados diretamente no S3 sem ingestão. Isso controla o custo de
armazenamento dentro do Redshift.

**Prazo estimado:** 4 semanas
*(Semanas 1–2: modelagem do schema dimensional — fatos de pedidos, dimensões de
rotas, produtos e datas; Semanas 3–4: ingestão de dados históricos, validação e
construção das primeiras queries analíticas)*

**Ganho estimado:** 40–50% de redução em custos de analytics em relação a ferramentas atuais

**Baseline e justificativa:** o baseline considera o uso atual de ferramentas como
Excel avançado com bases grandes, licenças de BI on-premises (Power BI Premium,
Tableau Server) ou instâncias EC2 usadas para processamento analítico ad hoc — padrão
comum em empresas deste porte sem infraestrutura cloud. O Redshift Serverless para
cargas intermitentes de analytics fica na faixa de USD 40–80/mês para o volume esperado
da FarmaHub. O ganho de 40–50% é conservador: se a empresa mantiver licenças de BI
locais, a economia pode ser maior, mas manter ao menos uma ferramenta de visualização
(como o Amazon QuickSight) ainda gera custo adicional que deve ser contabilizado.

---

## Conclusão

| Etapa | Serviços | Prazo | Ganho Estimado (Custos Computacionais) |
|---|---|---|---|
| 1 | S3 com tiering | 4 semanas | 30–35% em armazenamento |
| 2 | Lambda + EventBridge + SQS | 4 semanas | 50–60% em computação |
| 3 | Redshift Serverless | 4 semanas | 40–50% em analytics |

**Prazo total:** 12 semanas (migração faseada com validação entre etapas)

A projeção consolidada de redução de custos computacionais no primeiro ano situa-se
entre **35–45%**, considerando o perfil B2B intermediário-pequeno da FarmaHub. 

---

**Anexos**
- Diagrama de arquitetura AWS

<img width="1500" height="900" alt="Diagram" src="https://github.com/user-attachments/assets/33f37ab9-dd52-464b-b1f2-fd91af0bbe7c" />

---

*Assinatura do Responsável pelo Projeto:*
**Gabriel Veras**
