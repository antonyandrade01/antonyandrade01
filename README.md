<p align="center">
  <img src="https://github.com/antonyandrade01/.github/blob/main/assets/BannerPerfil2026.jpg?raw=true" alt="Antony Andrade - Backend Software Engineer | Python, PostgreSQL, Integrations">
</p>

<div align="center">

### Antony Andrade

<p><b>Backend Software Engineer | Python, PostgreSQL, Integrations</b></p>

<p>Working in healthcare software since 2021 across support, team supervision, QA, and backend development (LIS Unilab, ~1,000 clinical laboratories in Brazil). Outside work hours, I build and run my own products (Skorp CRM in beta with ~71k messages/month) and a Proxmox server. Product source code is private; each project below links to a public architecture showcase, and live demos are available on request.</p>

<p>
  <a href="https://www.linkedin.com/in/antonyandrade01/" target="_blank"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"/></a>
  <a href="mailto:antonymsandrade@gmail.com" target="_blank"><img src="https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"/></a>
  <img src="https://komarev.com/ghpvc/?username=antonyandrade01&label=Profile%20Views&color=blueviolet&style=for-the-badge" alt="Profile Views" />
</p>
</div>

<details align="center">
  <summary><b>🇧🇷 Clique aqui para ver a versão em Português</b></summary>

  ### Antony Andrade

  <p><b>Engenheiro de Software Backend | Python, PostgreSQL, Integrações</b></p>

  <p>Desde 2021 trabalho com software de saúde (LIS Unilab, cerca de 1.000 laboratórios no Brasil), passando por suporte, supervisão de equipe, QA e desenvolvimento backend. Em paralelo, construo e opero produtos próprios (Skorp CRM em beta com ~71 mil mensagens por mês) e um servidor Proxmox. O código dos produtos é privado; cada item abaixo aponta para uma vitrine pública de arquitetura, com demonstração sob pedido.</p>
</details>

---

### Real Architecture Decisions from Skorp CRM / Decisões Técnicas Reais do Skorp CRM

<div align="center">
<p><b>EN:</b> Three backend decisions from production code and incident fixes.<br><b>PT:</b> Três decisões de backend tiradas do código em produção e de correções reais.</p>
</div>

<details>
  <summary><b>1. PostgreSQL Row-Level Security with a restricted role / RLS por tenant com papel restrito</b></summary>
  <p>
    <b>EN:</b> PostgreSQL superusers bypass Row-Level Security even when it is forced on a table. The API and background workers connect with a restricted non-superuser role, while only the migration container connects as the table owner. Every request and job binds the active tenant in the database session, and a SQLAlchemy hook reapplies it whenever a mid-request commit opens a new transaction. Login and webhook lookup tables allow read queries before auth while keeping writes scoped by tenant.<br><br>
    <b>PT:</b> Superusuários ignoram o Row-Level Security no PostgreSQL mesmo quando a regra está forçada na tabela. A API e os workers conectam com um papel restrito sem privilégio de superusuário, enquanto apenas o container de migrations usa o papel dono das tabelas. Cada requisição e tarefa define o tenant ativo na sessão do banco, e um hook no SQLAlchemy reaplica essa variável quando um commit intermediário abre uma nova transação. Consultas de login e entrada de webhook liberam leitura antes da autenticação, mantendo escritas isoladas por tenant.
  </p>
</details>

<details>
  <summary><b>2. Channel-scoped Valkey locks and atomic release / Locks no Valkey por canal e liberação atômica</b></summary>
  <p>
    <b>EN:</b> Webhook deduplication keys include the channel ID alongside the external message ID. This fixed a production bug where two CRM tenants messaged each other: WhatsApp reuses the same message ID on both sides of a chat and across group members, so a global lock caused one tenant to drop messages. Contact creation uses a per-identity lock with a unique token checked via Lua script on release so a slow worker cannot delete another worker's lock after expiration. Valkey runs without key eviction so WhatsApp sessions and revoked tokens are never dropped silently.<br><br>
    <b>PT:</b> A chave de deduplicação de webhooks junta o ID do canal com o ID externo da mensagem. Isso corrigiu um bug em produção quando dois clientes do CRM conversavam entre si: o WhatsApp repete o mesmo ID nos dois lados da conversa e entre membros de grupo, então uma trava global fazia um tenant descartar mensagens. Na criação de contatos, a trava grava um token único e só é removida por script Lua se o token ainda for o mesmo, evitando que um worker lento apague a trava de outro após expirar. O Valkey roda sem despejo de chaves para nunca perder sessões do WhatsApp nem listas de tokens revogados.
  </p>
</details>

<details>
  <summary><b>3. RabbitMQ/Celery queues and strict lead merge / Filas RabbitMQ/Celery e merge só com prova exata</b></summary>
  <p>
    <b>EN:</b> WhatsApp (Baileys) events arrive through RabbitMQ with a dead-letter queue and explicit acknowledgment outside the error handler, preventing processed messages from going to the DLQ when a channel drops briefly. Celery workers limit their database connection pools per process so they do not exhaust PostgreSQL connections. For Meta Ads leads arriving with anonymous WhatsApp IDs (<code>@lid</code>), automatic merge only runs when WhatsApp sends both the anonymous ID and the real phone number together with a 1-to-1 match. Profile-picture hashing was removed from merge rules after tests showed shared company logos could merge unrelated contacts.<br><br>
    <b>PT:</b> Eventos do WhatsApp (Baileys) entram pelo RabbitMQ com fila de erro (DLQ) e confirmação fora do bloco de tratamento de erro, evitando mandar mensagens já salvas para a DLQ quando o canal oscila. Os workers do Celery limitam o pool de conexões por processo para não estourar o limite do PostgreSQL. No merge de leads vindos de anúncios da Meta (<code>@lid</code>), a unificação automática só ocorre quando telefone e identificador anônimo chegam juntos do WhatsApp com correspondência 1 para 1. O hash de foto de perfil foi retirado do critério porque testes mostraram que logotipos compartilhados podiam unir pessoas diferentes.
  </p>
</details>

---

### Projects / Projetos

> **EN:** Product source code is private. Each title links to a public architecture showcase with diagrams and technical decisions. Live demos are available on request.
>
> **PT:** O código dos produtos abaixo é privado. Cada título aponta para uma vitrine pública com diagramas de arquitetura e decisões técnicas. Demonstração ao vivo sob pedido.

#### **[Skorp CRM](https://github.com/antonyandrade01/skorp-crm)** · [skorp.com.br](https://skorp.com.br)
<p><b>EN:</b> Multi-tenant omnichannel CRM and sales pipeline in beta since August 2026. Connects 13 active channels (WhatsApp via Baileys and Cloud API, Instagram Direct, and website widget), handling ~71,000 messages per month. Built with Python, FastAPI, PostgreSQL (RLS), Valkey, RabbitMQ, Celery, SeaweedFS, and Next.js / React, with hourly encrypted backups and automated weekly restore tests.</p>
<p><b>PT:</b> CRM omnichannel e funil de vendas multi-tenant em beta desde agosto de 2026. Conecta 13 canais ativos (WhatsApp via Baileys e Cloud API, Instagram Direct e widget de site), com cerca de 71 mil mensagens por mês. Feito com Python, FastAPI, PostgreSQL (RLS), Valkey, RabbitMQ, Celery, SeaweedFS e Next.js / React, com backup cifrado de hora em hora e teste automático semanal de restauração.</p>
<b>Tech Stack:</b><br>
<img src="https://skillicons.dev/icons?i=python,fastapi,postgres,redis,rabbitmq,nextjs,ts,docker" />
<br clear="all"/>

<br>

<table width="100%">
<tr>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/skorp-fit">Skorp Fit</a></h4>
  <p align="center"><a href="https://gym.antonyandrade.com.br">gym.antonyandrade.com.br</a></p>
  <p><b>EN:</b> Offline-first strength training and GPS cardio PWA in daily use. Single TypeScript codebase with Node, Hono, React, PostgreSQL, and IndexedDB, plus Hevy import and a separate Python, Blender, and FFmpeg pipeline that generates 3D exercise animations.<br><br><b>PT:</b> PWA de treino de força e cardio por GPS que funciona offline, em uso no dia a dia. Base única em TypeScript com Node, Hono, React, PostgreSQL e IndexedDB, importação do Hevy e um pipeline separado em Python, Blender e FFmpeg para gerar animações 3D dos exercícios.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=ts,nodejs,react,vite,postgres,python,blender" /></p>
</td>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/skorp-clinic">Skorp Clinic</a></h4>
  <p align="center"><a href="https://evolucao.antonyandrade.com.br">evolucao.antonyandrade.com.br</a></p>
  <p><b>EN:</b> Multi-tenant clinical progress notes service in a pilot with health professionals. Receives WhatsApp audio or text over RabbitMQ, transcribes locally with faster-whisper, structures each session note separately from prior history, and lets clinicians review and approve it in a web panel. Built with Python, FastAPI, PostgreSQL (RLS), and RabbitMQ.<br><br><b>PT:</b> Serviço multi-tenant de evoluções clínicas em piloto com profissionais de saúde. Recebe áudio ou texto do WhatsApp via RabbitMQ, transcreve localmente com faster-whisper, estrutura a evolução do dia isolada do histórico e libera revisão e aprovação no painel web. Feito com Python, FastAPI, PostgreSQL (RLS) e RabbitMQ.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=python,fastapi,postgres,rabbitmq,docker,linux" /></p>
</td>
</tr>
<tr>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/skorp-finance">Skorp Financy</a></h4>
  <p align="center"><a href="https://financeiro.antonyandrade.com.br">financeiro.antonyandrade.com.br</a></p>
  <p><b>EN:</b> Personal and family finance app. Started in FastAPI and rewritten in Java 25 and Spring Boot 4 (Spring Modulith, Flyway, PostgreSQL RLS, Testcontainers) as a practical study project, with a Next.js web interface and bank statement import (OFX, CSV, PDF).<br><br><b>PT:</b> Sistema de controle financeiro pessoal e familiar. Começou em FastAPI e foi reescrito em Java 25 e Spring Boot 4 (Spring Modulith, Flyway, PostgreSQL com RLS e Testcontainers) como projeto prático de estudo, com interface web em Next.js e importação de extratos (OFX, CSV e PDF).</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=java,spring,postgres,nextjs,ts,docker" /></p>
</td>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/homelab">Homelab &amp; AI Operations Hub</a></h4>
  <p><b>EN:</b> Single-node Proxmox VE server running production VMs and LXC containers: Coolify with Traefik, Cloudflare Tunnel, SeaweedFS S3, a self-hosted GitHub Actions runner, Home Assistant, and Pi-hole. Also runs a personal AI hub that monitors servers, backups, and deploys over WhatsApp, backed by Hindsight memory, Groq/faster-whisper transcription, and a read-only Python MCP server.<br><br><b>PT:</b> Servidor Proxmox VE único com VMs e containers LXC para produção e uso pessoal: Coolify com Traefik, Cloudflare Tunnel, SeaweedFS S3, runner próprio do GitHub Actions, Home Assistant e Pi-hole. Roda também uma central de IA que acompanha servidores, backups e deploys pelo WhatsApp, com memória Hindsight, transcrição (Groq e faster-whisper reserva) e servidor MCP de leitura em Python.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=linux,docker,python,bash,githubactions,cloudflare" /></p>
</td>
</tr>
</table>

---

### Earlier Public Tooling / Ferramenta Pública Anterior

<table width="100%">
<tr>
<td width="100%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/training-automation-suite">Release Training Automation / Automação de Treinamentos de Versão</a></h4>
  <p><b>EN:</b> Built during my QA role at Uniware. Python CLI that queries MySQL, generates release training slide decks (<code>python-pptx</code>), and updates support tickets via the Movidesk REST API, cutting a full-day manual task down to a single command.<br><br><b>PT:</b> Feito na Uniware durante minha atuação como QA. CLI em Python que consulta MySQL, monta apresentações de treinamento de versão (<code>python-pptx</code>) e atualiza chamados via API do Movidesk, reduzindo uma tarefa manual de um dia inteiro para um comando.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=python,mysql" /></p>
</td>
</tr>
</table>

---

<h3 align="center">📊 GitHub Activity &amp; Stack / Atividade e Tecnologias 📊</h3>

<div align="center">
  <picture>
    <source
      media="(prefers-color-scheme: dark)"
      srcset="https://github-readme-activity-graph.vercel.app/graph?username=antonyandrade01&bg_color=0d1117&color=ffffff&line=00b4d8&point=ffffff&area=true&hide_border=true"
    >
    <source
      media="(prefers-color-scheme: light)"
      srcset="https://github-readme-activity-graph.vercel.app/graph?username=antonyandrade01&bg_color=fffeff&color=24292e&line=00b4d8&point=24292e&area=true&hide_border=true"
    >
    <img
      alt="Antony's Activity Graph"
      src="https://github-readme-activity-graph.vercel.app/graph?username=antonyandrade01&bg_color=fffeff&color=24292e&line=00b4d8&point=24292e&area=true&hide_border=true"
    >
  </picture>
</div>

<div align="center">
  <picture>
    <source
      media="(prefers-color-scheme: dark)"
      srcset="https://github-profile-trophy.vercel.app/?username=antonyandrade01&theme=discord&column=-1&no-frame=true&no-bg=true&margin-w=15&rank=-?"
    >
    <source
      media="(prefers-color-scheme: light)"
      srcset="https://github-profile-trophy.vercel.app/?username=antonyandrade01&theme=flat&column=-1&no-frame=true&no-bg=true&margin-w=15&rank=-?"
    >
    <img
      alt="Antony's GitHub Trophies"
      src="https://github-profile-trophy.vercel.app/?username=antonyandrade01&theme=flat&column=-1&no-frame=true&no-bg=true&margin-w=15&rank=-?"
    >
  </picture>
</div>

<div align="center">
  <a href="https://skillicons.dev">
    <img src="https://skillicons.dev/icons?i=python,fastapi,postgres,mysql,redis,rabbitmq,docker,linux,bash,ts,nextjs,githubactions,git&perline=13" />
  </a>
</div>

---

<h3 align="center">📫 Connect / Contato</h3>

<p align="center">
  <a href="https://www.linkedin.com/in/antonyandrade01/" target="_blank"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"/></a>
  <a href="mailto:antonymsandrade@gmail.com" target="_blank"><img src="https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"/></a>
  <a href="https://discordapp.com/users/TonyToninho" target="_blank"><img src="https://img.shields.io/badge/Discord-7289DA?style=for-the-badge&logo=discord&logoColor=white" alt="Discord"/></a>
</p>
