# Gerador de Podcast com IA: Roadmap e Arquitetura

Aplicativo mobile em **React Native** onde o usuário escolhe 1 entre 6 temas pré-definidos e recebe um podcast de \~2 minutos, com roteiro gerado por IA e voz sintetizada.

\---

## 1\. Visão geral da arquitetura

```
\[App React Native]  ──HTTP──▶  \[API Backend]  ──▶  \[LLM]  (gera o roteiro)
   Expo                           │            ──▶  \[TTS]  (gera o áudio)
                                  │
                                  ├──▶ \[Banco de dados]   (temas, episódios, status)
                                  └──▶ \[Armazenamento]    (arquivos .mp3)
```

### Componentes

|Componente|Tecnologia sugerida|Responsabilidade|
|-|-|-|
|App mobile|React Native + Expo|Exibir temas, acompanhar status e tocar o áudio|
|API backend|FastAPI (Python) ou Express (Node)|Orquestrar todo o pipeline|
|LLM|Claude, OpenAI ou similar|Gerar o roteiro a partir do prompt do tema|
|TTS|OpenAI TTS, Google Cloud TTS ou ElevenLabs|Converter o roteiro em voz|
|Banco de dados|SQLite (simples) ou Postgres|Temas, episódios e status|
|Armazenamento|Pasta local servida pela API, Supabase Storage ou Cloudflare R2|Guardar os `.mp3`|
|Hospedagem de teste|Docker Compose em Render, Railway, Fly.io ou VM gratuita|Rodar o backend|

Para a apresentação, também dá para rodar o backend local e expor com **ngrok** ou **Cloudflare Tunnel**.

\---

## 2\. Stack do app (React Native)

* **Expo** (managed workflow): setup rápido e testes em dispositivo real via Expo Go, sem precisar de Android Studio ou Xcode.
* **Expo Router** ou **React Navigation**: navegação entre as 3 telas.
* **TanStack Query (React Query)**: requisições e polling do status do episódio (`refetchInterval`).
* **expo-audio** (ou `expo-av` em versões mais antigas do Expo): reprodução do áudio.
* **TypeScript**: tipagem dos contratos da API.
* **Variável de ambiente** para a URL do backend (`EXPO\_PUBLIC\_API\_URL`), facilitando trocar entre local, túnel e servidor de teste.

### Estrutura de pastas sugerida

```
app/
├── (tabs ou stack)/
│   ├── index.tsx          # Tela de temas
│   ├── generating.tsx     # Tela de geração (polling)
│   └── player.tsx         # Tela do player
src/
├── api/                   # Cliente HTTP e funções (getThemes, createEpisode, getEpisode)
├── hooks/                 # useThemes, useEpisodePolling, useAudioPlayer
├── components/            # ThemeCard, ProgressSteps, PlayerControls
├── types/                 # Theme, Episode, EpisodeStatus
└── constants/             # Cores, textos
```

\---

## 3\. Pipeline de geração

1. O app envia o `theme\_id` escolhido.
2. A API cria um registro `episode` com status `pending` e devolve o `episode\_id` imediatamente.
3. Uma tarefa em segundo plano executa:

   * **Roteiro:** monta o prompt do tema e chama o LLM.
   * **Validação:** confere o tamanho do texto. Para 2 minutos de fala em português, mire em **260 a 300 palavras**. Se vier muito curto ou longo, regenera ou ajusta.
   * **Áudio:** envia o roteiro ao TTS e salva o `.mp3`.
   * **Medição:** lê a duração real do arquivo (ffprobe ou mutagen) e registra.
4. O status muda para `ready` (ou `failed`, com a mensagem de erro).
5. O app consulta o status periodicamente (**polling a cada 2 segundos**) e, quando `ready`, toca o áudio.

**Por que assíncrono com polling?** A geração leva de 15 a 40 segundos, longo demais para uma requisição síncrona. O polling evita timeout e permite mostrar etapas no app ("escrevendo roteiro", "gravando voz").

Para o escopo do trabalho, basta uma `BackgroundTask` do FastAPI ou uma fila simples em memória. Redis só se quiser demonstrar escalabilidade.

\---

## 4\. Endpoints

|Método|Rota|Função|
|-|-|-|
|GET|`/themes`|Lista os 6 temas (id, nome, ícone)|
|POST|`/episodes`|Recebe `theme\_id`, cria o episódio e inicia a geração|
|GET|`/episodes/{id}`|Retorna status, etapa atual, `audio\_url` e duração|
|GET|`/health`|Verificação de saúde do servidor|

### Exemplo de resposta de `GET /episodes/{id}`

```json
{
  "id": "a1b2c3",
  "theme\_id": 3,
  "status": "voicing",
  "audio\_url": null,
  "duration\_sec": null,
  "error": null
}
```

\---

## 5\. Modelo de dados

**themes**

|Campo|Descrição|
|-|-|
|`id`|Identificador|
|`name`|Nome exibido|
|`slug`|Identificador legível|
|`prompt\_template`|Prompt usado para gerar o roteiro|
|`icon`|Ícone do card|

**episodes**

|Campo|Descrição|
|-|-|
|`id`|Identificador|
|`theme\_id`|Tema escolhido|
|`status`|`pending`, `scripting`, `voicing`, `ready` ou `failed`|
|`script`|Texto do roteiro|
|`audio\_path`|Caminho do arquivo de áudio|
|`duration\_sec`|Duração real do áudio|
|`error`|Mensagem de erro, se houver|
|`created\_at`|Data de criação|

Guardar o `prompt\_template` na tabela de temas permite ajustar os prompts sem mexer no código.

\---

## 6\. Roteiro de desenvolvimento

### Etapa 1: Definição

* \[ ] Escolher os 6 temas (ex.: Tecnologia, História, Ciência, Negócios, Saúde, Curiosidades) e o tom do podcast.
* \[ ] Decidir entre um narrador ou dois apresentadores (um narrador é bem mais simples).
* \[ ] Escolher stack, LLM e TTS, e criar as chaves de API.

### Etapa 2: Prototipação do pipeline (sem app e sem API)

* \[ ] Script isolado: tema → roteiro → áudio.
* \[ ] Iterar nos prompts até o roteiro ficar bom e dentro de \~2 minutos.
* \[ ] Estrutura do roteiro: gancho de abertura, 2 a 3 pontos principais, fechamento curto.
* \[ ] Ajustar pontuação e texto para a fala soar natural no TTS.

### Etapa 3: Backend

* \[ ] Estruturar o projeto (rotas, serviços, modelos).
* \[ ] Implementar o banco, os endpoints e a tarefa em segundo plano.
* \[ ] Separar em módulos `script\_service` e `tts\_service`, para trocar de provedor facilmente.
* \[ ] Servir os arquivos de áudio via URL.
* \[ ] Guardar as chaves em variáveis de ambiente, nunca no app nem no repositório.

### Etapa 4: App React Native

* \[ ] Criar o projeto com Expo e configurar TypeScript.
* \[ ] **Tela de temas:** grade com os 6 cards, carregados de `GET /themes`.
* \[ ] **Tela de geração:** loading com a etapa atual, vinda do polling de `GET /episodes/{id}`.
* \[ ] **Tela do player:** play/pause, barra de progresso, botão "gerar outro".
* \[ ] Tratamento de erro: falha na geração, sem conexão, servidor fora do ar.
* \[ ] Testar em dispositivo real com Expo Go apontando para a URL do túnel ou do servidor de teste.

### Etapa 5: Estabilização

* \[ ] **Cache de episódios:** pré-gerar 2 ou 3 episódios por tema e entregar um deles na hora. Deixa a demonstração rápida e reduz custo.
* \[ ] **Rate limit** simples por IP ou dispositivo, para não estourar a cota das APIs.
* \[ ] **Retry** com backoff nas chamadas ao LLM e ao TTS.
* \[ ] **Logs** por etapa e tempo gasto em cada uma.

### Etapa 6: Deploy de teste e documentação

* \[ ] `Dockerfile` e `docker-compose.yml` para subir o backend com um comando.
* \[ ] Publicar num serviço gratuito ou rodar local com túnel.
* \[ ] README com arquitetura, como rodar e decisões técnicas.

\---

