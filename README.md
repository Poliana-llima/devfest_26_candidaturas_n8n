# Agente de IA para Busca e Gestão Inteligente de Candidaturas

Automação desenvolvida com **n8n**, **LLMs**, **Google Workspace** e **Telegram** para monitorar vagas, avaliar aderência com o perfil profissional do candidato e manter o humano no controle das decisões críticas.

O fluxo implementa uma arquitetura **Human-in-the-Loop**, em que a IA pode encontrar, analisar e preparar materiais de candidatura, mas a decisão de seguir ou não com uma oportunidade permanece com o usuário.

---

# Visão geral

O fluxo realiza, de forma automatizada:

1. Busca de vagas em múltiplas fontes.
2. Normalização dos dados recebidos.
3. Criação de ID determinístico para cada vaga.
4. Deduplicação contra vagas já processadas.
5. Avaliação da vaga por um Recruiter Agent.
6. Cálculo de score de compatibilidade.
7. Registro das oportunidades.
8. Notificação via Telegram.
9. Aprovação ou rejeição humana.
10. Geração de currículo e materiais personalizados.
11. Identificação do canal de candidatura.
12. Persistência do estado no Google Sheets.
13. Tratamento de retries e idempotência.

---

# Arquitetura

```text
We Work Remotely ─────┐
                      │
ProgramaThor ─────────┼─> Normalização
                      │
Google Alerts ────────┘
                           ↓
                 Normalize - Unified Jobs
                           ↓
                 Build Deterministic Job ID
                           ↓
                 Sheets - Load Existing Jobs
                           ↓
                      Filter New Jobs
                           ↓
                          Limit
                           ↓
                     Recruiter Agent
                           ↓
                  Combine Recruiter Result
                           ↓
                    IF Score >= 70
                      /          \
                   não            sim
                   ↓               ↓
                Arquiva         Telegram
                                   ↓
                              Humano decide
                              /           \
                          rejeita         aprova
                                           ↓
                                      Career Coach
                                           ↓
                                      Outreach LLM
                                           ↓
                              Currículo / Carta / E-mail
                                           ↓
                                    Google Workspace
                                           ↓
                                         CRM
```

---

# Tecnologias utilizadas

- n8n
- LLM / OpenAI-compatible model
- Google Sheets
- Google Docs
- Google Drive
- Telegram Bot API
- RSS
- HTTP Request
- JavaScript em Code Nodes

---

# Pré-requisitos

Antes de importar o workflow, tenha:

- uma instalação funcional do n8n;
- uma conta Google;
- credenciais OAuth2 ou Service Account configuradas no n8n;
- um bot do Telegram;
- uma credencial de LLM configurada no n8n;
- uma planilha Google Sheets que será utilizada como CRM;
- acesso à internet pelas instâncias do n8n.

O fluxo foi desenvolvido e validado em ambiente **n8n Self Hosted**.

---

# 1. Importando o workflow

Baixe o arquivo `.json` deste repositório.

No n8n:

```text
Workflows
→ Import from File
→ selecione o arquivo JSON
```

Após importar, **não execute o fluxo completo imediatamente**.

Primeiro configure todas as credenciais e parâmetros descritos neste README.

---

# 2. Configurando as credenciais

## Google

Crie ou selecione uma credencial Google compatível com:

- Google Sheets
- Google Drive
- Google Docs

No n8n:

```text
Credentials
→ New Credential
→ Google
```

Associe a credencial aos nodes Google presentes no workflow.

Dependendo da instalação, pode ser necessário configurar OAuth2 no Google Cloud Console.

---

## LLM

Configure a credencial usada pelos nodes de IA.

Exemplo:

```text
Credentials
→ OpenAI
```

ou outro provedor compatível com os nodes utilizados.

Associe a credencial a:

- Recruiter Agent
- Career Coach
- Basic LLM Chain - Outreach
- demais nodes de IA existentes no workflow

Não coloque API Keys diretamente dentro de prompts ou Code Nodes.

---

## Telegram

Crie um bot usando o **BotFather** no Telegram.

Comando:

```text
/newbot
```

O BotFather retornará um token.

Configure uma credencial Telegram no n8n:

```text
Credentials
→ Telegram API
```

Cole o token fornecido pelo BotFather.

---

# 3. Google Sheets — CRM

Crie uma planilha chamada, por exemplo:

```text
Controle_Candidaturas_IA
```

O workflow utiliza pelo menos três abas:

```text
Vagas
Logs
Config
```

---

# 4. Aba Vagas

Crie as seguintes colunas:

```text
ID
Data Encontrada
Empresa
Cargo
Local
Tipo
Link
Score IA
Motivos Match
Lacunas
Status Telegram
Data Aprovação
Currículo Gerado
Mensagem Gerada
Data Envio
Status Processo
Observações
Carta de Apresentação
Canal Candidatura
E-mail Candidatura
URL Candidatura
Instruções Candidatura
```

Não altere os nomes das colunas sem também revisar os nodes Google Sheets do workflow.

---

# 5. Aba Logs

Crie:

```text
timestamp
execution_id
event
job_id
details
```

Exemplos de eventos utilizados pelo workflow:

```text
Reprovada pela IA
Rejeitada pelo humano
Materiais gerados
Aprovada pelo humano
```

---

# 6. Aba Config

Estrutura:

```text
A1 = chave
B1 = valor
```

Na linha seguinte:

```text
A2 = CANDIDATE_PROFILE
B2 = perfil completo do candidato
```

---

# 7. Candidate Profile

O valor de `CANDIDATE_PROFILE` deve conter apenas fatos verdadeiros e verificáveis.

Exemplo de estrutura:

```text
Nome:
[SEU NOME]

Resumo profissional:
[SEU RESUMO]

Experiência profissional:

Empresa:
Cargo:
Período:
Responsabilidades:
Tecnologias:

Empresa:
Cargo:
Período:
Responsabilidades:
Tecnologias:

Formação:

Curso:
Instituição:
Status:

Tecnologias:
- JavaScript
- React
- Python
- etc.

Idiomas:

Certificações:

Projetos relevantes:
```

Evite colocar informações exageradas ou competências que você não possui.

Esse perfil será utilizado pelo Recruiter Agent e pelo Career Coach.

---

# 8. Configuração das fontes de vagas

## We Work Remotely

O fluxo utiliza RSS.

Verifique o node correspondente à fonte:

```text
Scout - We Work Remotely
```

ou nome equivalente.

Confirme se a URL RSS utilizada pelo workflow ainda está ativa.

---

# 9. ProgramaThor

O fluxo realiza uma requisição HTTP para:

```text
https://programathor.com.br/jobs-city/remoto
```

Node:

```text
Scout - ProgramaThor
```

Configuração:

```text
Method: GET
Authentication: None
Response Format: Text
Put Output in Field: data
```

O HTML retornado é tratado pelo node:

```text
Normalize - ProgramaThor
```

Esse node extrai:

```text
source
title
company
location
work_mode
company_size
seniority
contract
salary
technologies
link
description
external
published_at
```

---

# 10. Google Alerts

Crie um alerta em:

```text
Google Alerts
```

Sugestão de consulta:

```text
("Front-end Developer" OR "Web Developer" OR "JavaScript Developer" OR "React Developer")
("remoto" OR "remote")
```

No Google Alerts, selecione a saída em RSS.

Copie a URL gerada.

No node correspondente ao Google Alerts, substitua a URL RSS pelo seu feed.

Exemplo de node:

```text
Scout - Google Alerts
```

IMPORTANTE:

Desative:

```text
Always Output Data
```

Quando o Google Alerts não retornar vagas, o esperado é o node produzir:

```text
0 items
```

e não:

```json
{}
```

---

# 11. Normalize - Google Alerts

Esse node converte os diferentes campos recebidos pelo RSS para o schema interno do projeto.

Ele gera:

```json
{
  "source": "Google Alerts",
  "title": "",
  "company": "",
  "location": "",
  "work_mode": "",
  "company_size": "",
  "seniority": "",
  "contract": "",
  "salary": "",
  "technologies": [],
  "link": "",
  "description": "",
  "external": true,
  "published_at": null
}
```

Não é necessário preencher nada nesse node, salvo se o formato do Google Alerts mudar.

---

# 12. Normalize - Unified Jobs

Esse é um dos nodes mais importantes da arquitetura.

Ele padroniza todas as fontes para o mesmo schema:

```json
{
  "source": "",
  "title": "",
  "company": "",
  "location": "",
  "work_mode": "",
  "seniority": "",
  "contract": "",
  "salary": "",
  "company_size": "",
  "technologies": [],
  "link": "",
  "description": "",
  "external": false,
  "published_at": null
}
```

Todas as novas fontes adicionadas futuramente devem chegar a esse formato antes de continuar no workflow.

---

# 13. Build Deterministic Job ID

Esse node gera um identificador estável para cada vaga.

A prioridade é utilizar a URL da vaga.

Fluxo lógico:

```text
URL
↓
canonicalização
↓
hash
↓
ID
```

Também é realizada deduplicação dentro do próprio lote.

Isso evita que duas vagas com a mesma URL sejam processadas duas vezes na mesma execução.

Não é necessário preencher valores manuais nesse node.

---

# 14. Sheets - Load Existing Jobs

Configure esse node para ler a aba:

```text
Vagas
```

Use a planilha configurada anteriormente.

Em:

```text
Settings
```

mantenha:

```text
Execute Once = ON
```

Esse node carrega os IDs já processados.

---

# 15. Filter New Jobs

Esse node compara:

```text
IDs recebidos
VS
IDs já existentes no Sheets
```

Somente vagas ainda não existentes continuam no workflow.

Nenhuma configuração manual adicional é necessária.

---

# 16. Limit

O fluxo pode limitar quantas vagas serão processadas a cada execução.

Exemplo:

```text
Max Items: 3
```

Aumentar esse número aumenta:

- consumo de tokens;
- chamadas ao LLM;
- chamadas ao Google Sheets;
- tempo de execução.

Para testes, utilize:

```text
3
```

---

# 17. Recruiter Agent

O Recruiter Agent compara:

```text
perfil do candidato
VS
vaga
```

O perfil é carregado da aba:

```text
Config
```

através do node:

```text
Sheets - Load Candidate Profile - Scout
```

O Recruiter deve retornar:

```json
{
  "score": 75,
  "motivos": [],
  "lacunas": [],
  "recomendado": true
}
```

A regra final de recomendação não deve depender apenas do boolean do LLM.

O workflow aplica:

```text
recomendado = score >= 70
```

---

# 18. IF Score >= 70

A condição deve ser:

```text
{{ $json.score }}
```

Operador:

```text
is greater than or equal to
```

Valor:

```text
70
```

Resultado:

```text
score < 70
→ vaga não segue para aprovação humana

score >= 70
→ envia para Telegram
```

---

# 19. Telegram — Chat ID

Você precisa descobrir o seu Telegram Chat ID.

Envie uma mensagem para o bot.

Depois consulte as atualizações ou utilize um node Telegram Trigger para capturar:

```text
chat.id
```

Preencha o Chat ID nos nodes responsáveis pelo envio de mensagens.

Não utilize um Chat ID de exemplo presente no workflow.

---

# 20. Telegram Trigger Callback

O workflow utiliza botões para:

```text
Aprovar
Rejeitar
Retry
```

O callback contém o ID da vaga.

Exemplo conceitual:

```text
approve|JOB_ID
reject|JOB_ID
retry|JOB_ID
```

Após importar o workflow, ative o Telegram Trigger para registrar corretamente o webhook no Telegram.

---

# 21. Segurança do callback

Se o workflow possuir validação de usuário autorizado, informe o Chat ID ou User ID autorizado no node correspondente.

Isso evita que outros usuários possam acionar callbacks.

---

# 22. Human-in-the-Loop

Quando uma vaga recebe score suficiente:

```text
Recruiter
↓
Telegram
↓
Humano
```

Somente após:

```text
Aprovar
```

o workflow inicia a geração dos materiais.

Essa separação é proposital.

A IA recomenda.

O humano decide.

---

# 23. Callback e persistência

Depois que o usuário responde no Telegram, a execução original já pode ter terminado.

Por isso o callback realiza:

```text
Telegram Trigger Callback
↓
Parse Telegram Callback
↓
Sheets Load Candidate Profile Callback
↓
Sheets Lookup Job by ID
↓
Combine Callback + CRM Row
```

O estado do sistema não depende da memória da execução.

Ele é recuperado do CRM.

---

# 24. Idempotência

O workflow verifica se uma vaga já foi processada.

Exemplo de condições:

```text
Currículo Gerado != vazio
```

ou:

```text
Status Processo == Aprovada - gerando materiais
```

Isso evita gerar documentos várias vezes quando o usuário clica repetidamente em um botão.

---

# 25. Retry

O retry só deve ser permitido quando:

```text
Status Processo == Erro na geração de materiais
```

O fluxo deve impedir retry de processos concluídos com sucesso.

---

# 26. Career Coach

O Career Coach recebe:

```text
Perfil real do candidato
+
Descrição da vaga
```

e gera um currículo direcionado.

REGRA ESSENCIAL:

O modelo não pode inventar:

```text
empresa
cargo
período
projeto
tecnologia
certificação
resultado
métrica
skill
senioridade
```

Ele pode reorganizar e destacar informações verdadeiras do perfil.

---

# 27. Google Docs

Configure os nodes Google Docs com sua credencial Google.

Certifique-se de que a conta tenha permissão para criar arquivos no Google Drive.

O conteúdo do currículo é preparado pelo node:

```text
Prepare CV Formatting
```

e enviado para o Google Docs.

---

# 28. Basic LLM Chain - Outreach

Esse componente é responsável por gerar os materiais de comunicação da candidatura.

Ele recebe:

```text
perfil do candidato
+
vaga aprovada
```

e pode retornar:

```json
{
  "email": "",
  "cover_letter": "",
  "linkedin_message": "",
  "professional_summary": "",
  "application_channel": "",
  "application_email": "",
  "application_url": "",
  "application_instructions": ""
}
```

Valores aceitos para:

```text
application_channel
```

são:

```text
SITE
EMAIL
SITE_E_EMAIL
NAO_IDENTIFICADO
```

---

# 29. Parse Outreach Result

Mesmo solicitando JSON ao LLM, o workflow trata a resposta antes de utilizá-la.

O parser:

- remove blocos markdown;
- identifica o JSON;
- executa `JSON.parse`;
- valida campos;
- aplica valores default;
- impede respostas vazias.

Isso evita que uma resposta mal formatada do LLM quebre o restante da automação.

---

# 30. Google Sheets — mapeamento dos dados

Após a normalização, os campos internos utilizam nomes em inglês.

O Google Sheets continua utilizando nomes de colunas em português.

Exemplo de mapeamento:

```text
ID                ← id
Data Encontrada   ← data_encontrada
Empresa           ← company
Cargo             ← title
Local             ← location
Tipo              ← work_mode
Link              ← link
Score IA          ← score
Motivos Match     ← motivos
Lacunas           ← lacunas
```

Caso altere o schema interno, revise todos os nodes Google Sheets.

---

# 31. Campos de candidatura

O workflow também pode registrar:

```text
Carta de Apresentação
Canal Candidatura
E-mail Candidatura
URL Candidatura
Instruções Candidatura
```

Esses campos são gerados pelo componente Outreach.

---

# 32. Configurações que devem ser revisadas após importar

Verifique manualmente:

```text
[ ] Credencial do LLM

[ ] Credencial Google Sheets

[ ] Credencial Google Drive

[ ] Credencial Google Docs

[ ] Credencial Telegram

[ ] ID da planilha

[ ] Nome das abas

[ ] Chat ID do Telegram

[ ] Telegram User ID autorizado

[ ] RSS do Google Alerts

[ ] Candidate Profile

[ ] URLs externas

[ ] Limite de vagas por execução

[ ] Modelo de IA escolhido

[ ] Pasta de destino dos documentos
```

---

# 33. Ordem recomendada de testes

Não execute tudo de uma vez na primeira instalação.

Teste nesta sequência:

```text
1. We Work Remotely

2. ProgramaThor

3. Google Alerts

4. Normalize por fonte

5. Merge

6. Normalize - Unified Jobs

7. Build Deterministic Job ID

8. Sheets - Load Existing Jobs

9. Filter New Jobs

10. Limit

11. Recruiter

12. Combine Recruiter Result

13. IF Score >= 70

14. Gravação no Sheets

15. Telegram

16. Callback

17. Career Coach

18. Google Docs

19. Outreach

20. Atualização final do CRM
```

Só depois execute o workflow ponta a ponta.

---

# 34. Rate limits

Google Sheets possui limites de leitura e escrita.

Evite criar loops que executem diversas consultas individuais à planilha.

O projeto utiliza carregamento em lote sempre que possível.

Se receber:

```text
429
RESOURCE_EXHAUSTED
Quota exceeded
```

reduza a quantidade de chamadas e evite executar testes repetidos em sequência.

---

# 35. Segurança

Não suba para o GitHub:

```text
API Keys
Telegram Bot Token
OAuth Client Secret
credenciais Google
dados pessoais
Candidate Profile real
Chat IDs privados
e-mails privados
```

Antes de exportar o workflow, revise todos os nodes.

Prefira credenciais gerenciadas pelo próprio n8n.

---

# 36. Variáveis e dados de exemplo

Se publicar uma versão pública do workflow, substitua informações reais por placeholders.

Exemplo:

```text
YOUR_TELEGRAM_CHAT_ID
YOUR_GOOGLE_SHEET
YOUR_GOOGLE_ALERTS_RSS
YOUR_CANDIDATE_PROFILE
YOUR_GOOGLE_DRIVE_FOLDER
```

---

# 37. Testando a deduplicação

Execute o Scout uma primeira vez.

Depois que as vagas forem persistidas no Sheets, execute novamente.

O esperado é:

```text
Filter New Jobs
→ 0 items
```

quando nenhuma nova vaga existir.

---

# 38. Testando Human-in-the-Loop

Para testar sem depender de uma vaga real com score alto, utilize temporariamente um registro de teste.

Evite alterar permanentemente o threshold de produção.

Teste:

```text
vaga
↓
Telegram
↓
Aprovar
↓
callback
↓
lookup
↓
Career Coach
```

---

# 39. Status sugeridos

Alguns estados utilizados pelo fluxo:

```text
Aprovada - gerando materiais
Erro na geração de materiais
CONFIRMADO
REVISAO_HUMANA
```

Mantenha os valores utilizados pelas condições do workflow.

Alterar textos de status pode quebrar IFs e regras de retry.

---

# 40. Princípios de arquitetura

Este projeto segue alguns princípios importantes:

## LLM não é regra de negócio

Utilize IA para:

```text
interpretação
classificação
resumo
comparação semântica
geração textual
```

Utilize código para:

```text
thresholds
IDs
deduplicação
validação
idempotência
estado
roteamento
```

---

## Human-in-the-Loop

A IA pode recomendar uma ação.

A decisão crítica continua com o humano.

---

## Estado persistente

Não dependa da execução anterior do n8n.

Persistir estado permite:

- callbacks;
- workflows assíncronos;
- retries;
- auditoria;
- recuperação após falhas.

---

## Saída estruturada não significa saída confiável

Mesmo pedindo JSON ao LLM:

```text
valide
normalize
parse
trate erro
```

antes de utilizar os dados.

---

# 41. Troubleshooting

## Google Alerts retorna `[{}]`

Verifique se:

```text
Always Output Data
```

está habilitado.

Desative.

Feed sem resultados deve retornar:

```text
0 items
```

---

## Telegram retorna `can't parse entities`

Remova o Parse Mode ou envie mensagem em texto simples.

---

## Google Sheets retorna 429

Erro:

```text
Quota exceeded for quota metric 'Read requests'
```

Reduza chamadas ao Sheets e evite leitura dentro de loops.

---

## Vagas antigas aparecem como novas

Verifique:

```text
Build Deterministic Job ID
```

Não altere a regra de geração de IDs após já ter registros persistidos.

---

## Vagas duplicadas dentro da mesma execução

Verifique se o node de geração de ID também realiza deduplicação interna através de um `Set`.

---

## LLM retorna JSON inválido

Verifique o node:

```text
Parse Outreach Result
```

ou parser equivalente.

Nunca utilize diretamente uma resposta textual de LLM em nodes críticos sem validação.

---

# 42. Expansão para novas fontes

Para adicionar uma nova fonte:

```text
Nova Fonte
↓
Normalize - Nova Fonte
↓
Merge
↓
Normalize - Unified Jobs
```

A nova fonte deve produzir ou ser convertida para:

```json
{
  "source": "",
  "title": "",
  "company": "",
  "location": "",
  "work_mode": "",
  "seniority": "",
  "contract": "",
  "salary": "",
  "company_size": "",
  "technologies": [],
  "link": "",
  "description": "",
  "external": false,
  "published_at": null
}
```

Depois disso, o restante da arquitetura permanece independente da origem da vaga.

---

# 43. Importante sobre sites de vagas

Verifique os termos de uso, políticas de acesso e limites de cada plataforma antes de implementar scraping ou automação.

Dê preferência a:

- RSS;
- APIs oficiais;
- páginas públicas;
- feeds autorizados.

---

# Objetivo do projeto

Este projeto demonstra que construir agentes de IA confiáveis envolve mais do que conectar um LLM a um workflow.

Os principais desafios estão em:

```text
normalização
deduplicação
saída estruturada
persistência de estado
rate limits
callbacks
idempotência
retries
Human-in-the-Loop
```

O LLM é apenas um componente do sistema.

---

# Autora

**Poliana Lima**

Projeto apresentado no:

**DevFest Lauro de Freitas 2026**

Tema:

**Do currículo à candidatura: construindo um agente de IA com n8n, LLMs e Human-in-the-Loop**

---

# Licença

Defina a licença desejada para o projeto antes da publicação.

Para projetos públicos e reutilizáveis, uma opção comum é:

```text
MIT License
```

Adicione o arquivo `LICENSE` ao repositório caso deseje permitir reutilização.
