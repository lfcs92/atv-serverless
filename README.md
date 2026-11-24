<div align="center">

# ⚡ Atividade Individual 1 – Serverless

**Infraestrutura de Software · ADS · CESAR School · 2026.2**

![AWS SAM](https://img.shields.io/badge/AWS%20SAM%20CLI-1.174.0-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-29.9.3-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Caminho](https://img.shields.io/badge/Caminho-A%20(SAM%20local)-2EA44F?style=for-the-badge)

👤 **Aluno:** Luís Felipe Carneiro da Silva

🎥 **Vídeo (até 3 min):** [https://youtu.be/dp7sFIO2tmk](https://youtu.be/dp7sFIO2tmk)

</div>

<div align="justify">

A função `lambda_handler` recebe por HTTP (`POST /guias`) uma requisição de hemocomponentes e devolve a **guia de transporte**. Com ela observei o ciclo de vida do ambiente de execução (cold start e reaproveitamento), a falta de garantia de estado entre chamadas e o limite de tempo de execução. Usei o **Caminho A (AWS SAM CLI local)**, em Python 3.12, sem conta na AWS.

</div>

---

## 🧰 1. Ambiente

| Item | Versão |
|---|---|
| 🟠 AWS SAM CLI | 1.174.0 |
| 🔵 Docker | 29.9.3 (Docker Desktop, Windows) |
| 🐍 Runtime da função | Python 3.12 (imagem `public.ecr.aws/lambda/python:3.12-rapid-x86_64`) |
| 💻 Terminal | PowerShell no terminal do VS Code (Windows) |

📸 Evidências: `evidencias/01-versoes-docker-sam.png` e `evidencias/02-sam-init.png`.

## 📁 2. Estrutura do repositório

```
template.yaml          # SAM: função HelloWorldFunction, POST /guias, Timeout em Globals
samconfig.toml         # configuração do SAM (inclui warm_containers)
hello_world/app.py     # handler que gera a guia de transporte
events/pedido.json     # evento no formato do API Gateway (gerado com sam local generate-event)
tests/                 # testes gerados pelo sam init (não usados nesta atividade)
evidencias/            # prints dos comandos e das saídas
```

## ▶️ 3. Como rodar

<div align="justify">

Pré-requisitos: Docker Desktop aberto ("Engine running"), AWS SAM CLI e Python 3.12 (ou o build em container, ver abaixo).

</div>

```powershell
# 1. Gerar o evento (PowerShell; Out-File evita UTF-16 do operador >)
sam local generate-event apigateway aws-proxy --method POST --path guias --body '{\"requisicaoId\":\"R-001\",\"hospital\":\"H-Recife\",\"bolsas\":[\"O-NEG-0142\"]}' | Out-File -Encoding ascii events\pedido.json
# No arquivo gerado, o body vem em base64 ("isBase64Encoded": true).
# Troquei o body pelo JSON em texto puro e "isBase64Encoded" para false (ver seção 6).

# 2. Build (usei --use-container porque não tenho o Python 3.12 instalado localmente)
sam build --use-container

# 3. Invocar a função com o evento (rodar duas vezes e anotar o tempo)
sam local invoke -e events\pedido.json

# 4. Acionar por HTTP (terminal 1)
sam local start-api
# terminal 2, três vezes:
curl.exe -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{\"requisicaoId\":\"R-002\",\"hospital\":\"H-Caruaru\"}'

# 4b. Repetir com containers reaproveitados
sam local start-api --warm-containers LAZY

# 5. Limite de tempo (Timeout 3 em Globals no template.yaml)
curl.exe -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{\"requisicaoId\":\"R-003\",\"demorar\":true}'
# Depois: Timeout: 10 no template.yaml, sam build --use-container, subir o start-api e repetir.
```

> [!TIP]
> Em Git Bash/WSL o `curl` funciona no lugar do `curl.exe` e o JSON usa aspas simples sem escape. No PowerShell, `curl` é um apelido de outro comando, por isso usei `curl.exe`.

## 🛠️ 4. O que fiz, passo a passo

<div align="justify">

1. **Ambiente e projeto.** Instalei o SAM CLI e o Docker, conferi as versões e rodei `sam init` (Quick Start → Hello World Example → pacote zip, projeto `atv-serverless`). No `template.yaml` troquei o caminho `/hello` por `/guias` e o método `get` por `post`, deixei o `Timeout` de `Globals` em 3 s e substituí o `hello_world/app.py` pelo código da atividade.
2. **Invocação com evento.** Gerei o evento, fiz o build e rodei `sam local invoke -e events\pedido.json` duas vezes, depois uma terceira vez com `Measure-Command` para medir o tempo total.
3. **Gatilho HTTP.** Subi o `sam local start-api` e fiz três chamadas com `curl.exe` em três configurações: com `warm_containers = "EAGER"` (valor que o template do `sam init` já trazia no `samconfig.toml`), sem reaproveitamento (linha comentada) e com `--warm-containers LAZY`.
4. **Experimentos.** Enviei `demorar: true` com `Timeout: 3`, registrei o erro, aumentei o `Timeout` para 10, refiz o build e repeti.

</div>

## 📊 5. Resultados medidos

### 5.1 Invocação com evento (`sam local invoke`)

| Execução | Init Duration | Duration | Billed Duration | `invocacaoNesteAmbiente` | Container |
|---|---|---|---|---|---|
| 1ª | 0,11 ms | 65,50 ms | 66 ms | 1 | `0f5d678a…` |
| 2ª | 0,08 ms | 149,71 ms | 150 ms | 1 | `d9fdea39…` |
| 3ª (com `Measure-Command`) | 0,06 ms | 171,54 ms | 172 ms | 1 | `1fd50c3a…` |

- ⏱️ Tempo **total** do comando na 3ª execução: **15,41 s** (`TotalSeconds: 15,4141011`), contra 171,54 ms de execução da função.
- 📦 Na 1ª execução o SAM mostrou `Local image was not found` e `Building image...` antes de invocar.
- ✅ Resposta: `statusCode 201` com `{"guia": "G-R-001", "hospital": "H-Recife", "bolsas": ["O-NEG-0142"], "invocacaoNesteAmbiente": 1}`.

📸 Evidências: `03-invoke-1a-execucao.png`, `04-invoke-2a-execucao.png`, `05-start-api-eager-1a-chamada.png` (mostra o `TotalSeconds` no topo).

### 5.2 Gatilho HTTP e reaproveitamento do ambiente (`sam local start-api`)

| Cenário | `invocacaoNesteAmbiente` (3 chamadas) | Duration (3 chamadas) | Containers |
|---|---|---|---|
| 🟢 `warm_containers = "EAGER"` (padrão do template) | **1, 2, 3** | 138,07 / 1,21 / 1,44 ms | um só (`d0303e2a…`), criado ao subir o servidor |
| 🔴 Sem reaproveitamento (linha comentada) | **1, 1, 1** | 143,08 / 64,91 / 59,23 ms | um novo por chamada (`d330702c…`, `d33af1e1…`, `e3bb25f7…`) |
| 🟢 `--warm-containers LAZY` | **1, 2, 3** | 140,92 / 1,27 / 2,36 ms | um só (`23a3b768…`), criado na 1ª chamada |

<div align="justify">

Nos cenários com reaproveitamento o log mostra `Lambda function 'HelloWorldFunction' is already running` nas chamadas 2 e 3. No cenário sem reaproveitamento aparece um novo bloco `Mounting … SAM_CONTAINER_ID` a cada chamada.

</div>

📸 Evidências: `05`, `06` e `07` (EAGER), `08-start-api-sem-warm-3-curls.png`, `09-start-api-lazy-3-curls.png`, `10-samconfig-warm-containers.png`.

### 5.3 Limite de tempo

| Configuração | Resposta ao `curl` | Log do SAM | Duration |
|---|---|---|---|
| ❌ `Timeout: 3` e `demorar: true` | HTTP 502, `{"message":"Internal server error"}` | `Function 'HelloWorldFunction' timed out after 3 seconds` e `Invalid lambda response received: Lambda response must be valid json` | 3000,00 ms (cobrado: 3000 ms) |
| ✅ `Timeout: 10` e `demorar: true` | HTTP 201, `{"guia": "G-R-003", "hospital": null, "bolsas": [], "invocacaoNesteAmbiente": 1}` | `REPORT … Duration: 5141.86 ms` | 5141,86 ms (cobrado: 5142 ms) |

📸 Evidências: `11-timeout-3s.png` e `12-timeout-10s.png`.

## 🚧 6. Problemas encontrados e como resolvi

| ⚠️ Problema | ✅ Solução |
|---|---|
| **Runtime 3.15 no `sam init`.** Ao aceitar o "runtime mais popular", o template saiu com `python3.15`. | Troquei para `python3.12`, como pede o enunciado. |
| **Python 3.12 não instalado no PC.** O `sam build` falhou com `PythonPipBuilder:Validation – Binary validation failed for python` (eu só tinha o Python 3.13). | Usei `sam build --use-container`, que faz o build dentro do Docker. |
| **Git Bash.** O `sam local generate-event` falhou com `'C:\Program' não é reconhecido…`, por causa do espaço em `C:\Program Files`. | Passei a usar o PowerShell para os comandos `sam`. |
| **Body em base64.** O SAM 1.174.0 gerou o evento com `"isBase64Encoded": true`, e o handler faz `json.loads(event["body"])`. | Editei o `events/pedido.json`: JSON em texto puro e `"isBase64Encoded": false`. |
| **`samconfig.toml` com `EAGER`.** O template já trazia `warm_containers = "EAGER"` em `[default.local_start_api.parameters]`, então o `start-api` "normal" já reaproveitava o container (`1, 2, 3`). | Comentei essa linha para ter o cenário sem reaproveitamento. Os testes de timeout dos prints rodaram em modo EAGER (o log mostra `Initializing the lambda functions containers`), o que não altera o resultado, que depende só do `Timeout` e do `sleep(5)`. |

---

## 💬 7. Respostas

### 1️⃣ Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?

<div align="justify">

Na primeira invocação o SAM ainda teve de preparar a imagem do Docker (`Local image was not found`) e subir o container. O `Duration` reportado pela função não seguiu o padrão esperado (65,50 ms na 1ª, 149,71 ms na 2ª e 171,54 ms na 3ª), porque o `sam local invoke` cria um container novo a cada chamada e o Docker no Windows varia bastante. O custo real de subir o ambiente apareceu no tempo total do comando: 15,4 s, contra 172 ms do handler.

Na fase de **Init**, antes do handler, o Lambda cria o ambiente de execução, baixa o código, inicia o runtime Python e executa o código fora do handler (imports e `contador = 0`). Esse custo na primeira chamada de um ambiente novo é o **cold start**. Com o ambiente reaproveitado, o Init não se repete: no `start-api` com containers reaproveitados a 1ª chamada levou cerca de 140 ms e as seguintes 1 a 2 ms.

</div>

### 2️⃣ O que `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre estado em variável global?

<div align="justify">

Com o ambiente reaproveitado (`EAGER` e `LAZY`) o campo valeu **1, 2, 3**. Sem reaproveitamento valeu **1, 1, 1**, porque cada chamada subiu um container novo, com o `contador` começando em 0 (o `SAM_CONTAINER_ID` mudou a cada vez).

Uma variável global só guarda estado enquanto o ambiente for o mesmo, e o Lambda não garante isso: ele pode criar ou descartar ambientes a qualquer momento, e vários rodam em paralelo, cada um com o seu contador. Estado que precisa ser correto deve ficar em um serviço externo (banco, cache ou S3).

</div>

### 3️⃣ O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda e o que fazer com um trabalho mais longo?

<div align="justify">

Com `Timeout: 3` e `sleep(5)`, a função foi interrompida aos 3 s (`timed out after 3 seconds`, `Duration: 3000.00 ms`) e o `curl` recebeu HTTP 502 com `{"message":"Internal server error"}`. Com `Timeout: 10`, a mesma requisição voltou 201 após 5141,86 ms.

O tempo máximo de uma execução no Lambda é de **15 minutos (900 s)**. Para trabalhos mais longos, dá para dividir em partes menores, usar uma fila (SQS), orquestrar com Step Functions ou mover para um serviço sem esse limite, como ECS/Fargate ou AWS Batch.

</div>

### 4️⃣ Quanto custa um milhão de execuções desta função com 128 MB e 100 ms (us-east-1)?

- 🧾 Requisições: 1.000.000 ÷ 1.000.000 × US$ 0,20 = **US$ 0,20**
- ⚙️ Computação: 0,125 GB × 0,1 s = 0,0125 GB-s por execução; × 1.000.000 = 12.500 GB-s; × US$ 0,0000166667 = **US$ 0,2083**
- 💰 **Total ≈ US$ 0,41** (dentro da faixa gratuita mensal, US$ 0).

### 5️⃣ Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

<div align="justify">

✅ **Faria sentido** em tarefas curtas, sem estado e acionadas por evento: gerar a guia de transporte quando uma entrega é criada, tratar um alerta de temperatura da cadeia fria, avisar por rotina agendada sobre bolsas perto do vencimento e notificar doadores.

❌ **Não faria sentido** no processamento de pedidos e no roteamento (`ProcessadorPedidoService`): eles dependem de índice de estoque, fila FEFO e grafo mantidos em memória e de transação no banco. Numa função esse estado se perderia a cada ambiente novo, e o cold start de uma aplicação Spring Boot é pesado.

</div>

---

## 🎥 8. Vídeo

<div align="justify">

Gravação de tela mostrando a invocação com evento, a chamada por HTTP e o erro de tempo limite: [https://youtu.be/dp7sFIO2tmk](https://youtu.be/dp7sFIO2tmk)

</div>

## 🤖 9. Uso de IA

Conforme a PO-CESAR-021:

<div align="justify">

- **Ferramenta:** Claude (Anthropic).
- **Finalidade:** orientação, esqueleto do repositório, diagnóstico de erros de ambiente (Git Bash, Python 3.12, evento em base64, `warm_containers`) e rascunho do README.
- **Como validei:** rodei todos os comandos no meu computador, conferi os resultados contra os logs, refiz a conta de custo e revisei as respostas.

</div>
