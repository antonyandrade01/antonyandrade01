<p align="center">
  <img src="https://github.com/antonyandrade01/.github/blob/main/assets/BannerPerfil.png?raw=true" alt="Antony Andrade - Backend Software Engineer | Python, PostgreSQL, Integrations">
</p>

<div align="center">

### Antony Andrade

<p><b>Backend Software Engineer | Python, PostgreSQL, Integrations</b></p>

<p>Working in healthcare software since 2021 across support, team supervision, QA, and backend development (LIS Unilab, ~1,000 clinical laboratories in Brazil). Outside work hours, I build and run my own software products (Skorp CRM in beta production with 71k messages/30 days) and a self-hosted Proxmox server. Product repositories are private; each project below links to a public architecture showcase, and live demos are available on request.</p>

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

  <p>Desde 2021 em software de saúde (LIS Unilab, cerca de 1.000 laboratórios clientes): suporte, supervisão de equipe, QA e desenvolvimento backend. Em paralelo, construo e opero produtos próprios em produção (Skorp CRM em beta, com 71 mil mensagens em 30 dias) e um servidor Proxmox com VMs e containers LXC. O código dos produtos é privado; cada item abaixo aponta para uma vitrine pública de arquitetura, com demonstração sob pedido.</p>
</details>

---

### Real Architecture Decisions from Skorp CRM / Decisões Técnicas Reais do Skorp CRM

<div align="center">
<p><b>EN:</b> Three backend decisions pulled directly from production code and incident postmortems.<br><b>PT:</b> Três decisões de backend tiradas direto do código em produção e de incidentes reais.</p>
</div>

<details>
  <summary><b>1. PostgreSQL Row-Level Security with a dedicated non-superuser role / RLS por tenant com papel não-superusuário</b></summary>
  <p>
    <b>EN:</b> Superusers bypass PostgreSQL RLS even with <code>FORCE ROW LEVEL SECURITY</code>, so the API, Celery workers, and beat connect as a restricted role (<code>crm_app_user</code>) while only the one-shot Alembic migration container uses the owner role. Every session sets <code>app.current_tenant_id</code> via <code>SELECT set_config('app.current_tenant_id', :tid, true)</code>, and a SQLAlchemy <code>after_begin</code> listener reapplies it automatically whenever a new transaction opens after a mid-request <code>commit()</code>. Tables with unauthenticated lookups (<code>users</code> on login, <code>channels</code> on inbound webhooks) allow permissive <code>SELECT</code> with strict tenant checks on writes.<br><br>
    <b>PT:</b> Superusuário ignora RLS no PostgreSQL mesmo com <code>FORCE ROW LEVEL SECURITY</code>. Por isso a API, o Celery worker e o beat conectam com um papel restrito (<code>crm_app_user</code>), enquanto só o container one-shot de migrations usa o papel dono. Cada sessão aplica <code>SELECT set_config('app.current_tenant_id', :tid, true)</code> e um listener <code>after_begin</code> no SQLAlchemy reaplica a variável quando uma nova transação abre após um <code>commit()</code> no meio da requisição. Tabelas consultadas antes da autenticação (<code>users</code> no login e <code>channels</code> em webhooks) liberam <code>SELECT</code> e isolam escritas por tenant.
  </p>
</details>

<details>
  <summary><b>2. Channel-scoped Valkey locks &amp; Lua ownership release / Locks no Valkey por canal e liberação atômica em Lua</b></summary>
  <p>
    <b>EN:</b> Inbound webhooks deduplicate using atomic <code>SET NX EX 60</code> on <code>lock:{channel.id}:{message_type}:{external_id}</code>. Scoping by <code>channel.id</code> fixed a real production bug: WhatsApp reuses the same <code>key.id</code> on both sides of a chat and across group members, so a global lock caused one tenant to drop messages when two tenants messaged each other. Contact creation uses a separate per-identity lock with a unique UUID token released through a Lua compare-and-delete script, preventing a slow worker whose TTL expired from deleting another worker's lock. Valkey runs with <code>maxmemory-policy noeviction</code> so WhatsApp session state and JWT revocation lists are never silently evicted.<br><br>
    <b>PT:</b> A deduplicação de webhooks usa <code>SET NX EX 60</code> atômico na chave <code>lock:{channel.id}:{message_type}:{external_id}</code>. Incluir o <code>channel.id</code> corrigiu um bug real: o <code>key.id</code> do WhatsApp é igual nos dois lados de uma conversa e entre membros de um grupo, então uma trava global fazia um tenant descartar mensagens quando dois clientes do CRM conversavam entre si. Na criação de contatos, a trava grava um token UUID e só é removida por script Lua de compare-and-delete, evitando que um worker lento apague a trava de outro após o TTL. O Valkey roda com <code>noeviction</code> para nunca despejar sessão do WhatsApp nem blacklist de JWT.
  </p>
</details>

<details>
  <summary><b>3. RabbitMQ/Celery queues &amp; proof-only lead merge / Filas RabbitMQ/Celery e merge de leads só com prova exata</b></summary>
  <p>
    <b>EN:</b> Evolution API events enter through RabbitMQ (<code>aio-pika</code>, <code>prefetch_count=30</code>, DLX/DLQ, fixed node hostname so deploys do not orphan broker state) with <code>ack()</code> kept outside the <code>reject(requeue=False)</code> error handler. Celery runs 8 prefork workers with <code>task_acks_late=True</code> and per-process SQLAlchemy pool limits (<code>3+2</code> per child) to stay under PostgreSQL <code>max_connections</code>. For Meta Ads WhatsApp <code>@lid</code> leads, automatic contact merge only runs when WhatsApp delivers both the LID and the real phone together and the match is strictly 1-to-1 (handling Brazilian DDI 55 and 9th-digit variants). Avatar hashing (<code>profile_pic_hash</code>) was explicitly removed as a merge signal after testing showed shared company logos could merge unrelated people. Merged duplicates are soft-deleted with <code>merged_into_id</code>.<br><br>
    <b>PT:</b> Eventos da Evolution API entram via RabbitMQ (<code>aio-pika</code>, <code>prefetch_count=30</code>, DLX/DLQ, hostname fixo no nó para o deploy não criar nós órfãos), mantendo o <code>ack()</code> fora do bloco que envia para a DLQ. O Celery roda com 8 workers prefork, <code>task_acks_late=True</code> e pool reduzido por filho (<code>3+2</code>) para não estourar o <code>max_connections</code> do PostgreSQL. No merge de leads vindos de anúncios da Meta (<code>@lid</code>), a unificação automática só acontece quando telefone e LID chegam juntos do WhatsApp com correspondência 1 para 1 (incluindo DDI 55 e nono dígito). O hash de foto de perfil (<code>profile_pic_hash</code>) foi retirado do critério de merge porque logos compartilhadas podiam unir pessoas diferentes. Duplicatas vão para soft delete com <code>merged_into_id</code>.
  </p>
</details>

---

### Projects / Projetos

> **EN:** Source code for the products below is private. Each title links to a public showcase repository with architecture diagrams and technical decisions. Live demonstrations are available on request.
>
> **PT:** O código dos produtos abaixo é privado. Cada título aponta para uma vitrine pública com diagramas de arquitetura e decisões técnicas. Demonstração ao vivo sob pedido.

#### **[Skorp CRM](https://github.com/antonyandrade01/skorp-crm) (Featured / Destaque)**
<p><b>EN:</b> Multi-tenant omnichannel CRM and sales pipeline in beta production since August 2026. Running 13 connected channels (WhatsApp via Evolution API and Cloud API, Instagram Direct, website widget), ~3,000 contacts, and 71,000 messages over 30 days (~2,400/day). Built with Python 3.14, FastAPI, PostgreSQL 18 (RLS, 100 Alembic migrations), Valkey 9, RabbitMQ 4, Celery 5.6, SeaweedFS (S3), and Next.js 16 / React 19. Backed by 102 Pytest files on the backend and 17 frontend unit suites, plus hourly encrypted backups and automated weekly restore tests. Expo mobile app, Google Meet extension, Telegram, and VoIP are in testing.</p>
<p><b>PT:</b> CRM omnichannel multi-tenant em produção beta desde agosto de 2026. Opera 13 canais conectados (WhatsApp via Evolution API e Cloud API, Instagram Direct e widget de site), cerca de 3 mil contatos e 71 mil mensagens em 30 dias (~2.400/dia). Construído com Python 3.14, FastAPI, PostgreSQL 18 (RLS, 100 migrations Alembic), Valkey 9, RabbitMQ 4, Celery 5.6, SeaweedFS (S3) e Next.js 16 / React 19. Conta com 102 arquivos de teste Pytest no backend, 17 suítes unitárias no frontend, backup cifrado de hora em hora e teste automático de restauração semanal. App Expo, extensão do Google Meet, Telegram e VoIP seguem em teste.</p>
<b>Tech Stack:</b><br>
<img src="https://skillicons.dev/icons?i=python,fastapi,postgres,redis,rabbitmq,nextjs,ts,docker" />
<br clear="all"/>

<br>

<table width="100%">
<tr>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/skorp-fit">Skorp Fit</a></h4>
  <p><b>EN:</b> Offline-first workout and GPS cardio PWA in production (9 users, 220 logged workouts). Single TypeScript codebase on Node 24 (native type stripping) + Hono 4.7 + React 19 + PostgreSQL JSONB + IndexedDB, with Hevy import and a separate Python + Blender 4.2 headless pipeline for 320 CC0 3D exercise animations.<br><br><b>PT:</b> PWA offline-first de treinos e cardio por GPS em produção (9 usuários, 220 treinos registrados). Base única em TypeScript no Node 24 (type stripping nativo), Hono 4.7, React 19, PostgreSQL com JSONB e IndexedDB, importação do Hevy e pipeline separado em Python + Blender 4.2 headless para 320 animações 3D CC0.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=ts,nodejs,react,vite,postgres,python,blender" /></p>
</td>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/skorp-clinic">Skorp Clinic</a></h4>
  <p><b>EN:</b> Multi-tenant clinical progress notes service in early testing (2 users, 3 clinicians, 10 notes; 103 Pytest tests passing). Consumes WhatsApp audio/text via RabbitMQ, transcribes locally with faster-whisper, isolates session structuring from patient history, enforces strict RLS (no DELETE for app role), and queues post-approval consolidation via SQL trigger + <code>SKIP LOCKED</code>.<br><br><b>PT:</b> Serviço multi-tenant de evoluções clínicas em teste inicial (2 usuários, 3 profissionais, 10 evoluções; 103 testes Pytest passando). Consome áudio/texto do WhatsApp via RabbitMQ, transcreve localmente com faster-whisper, isola o ato clínico do histórico, aplica RLS sem DELETE no papel da API e consolida após aprovação por trigger SQL + <code>SKIP LOCKED</code>.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=python,fastapi,postgres,rabbitmq,docker,linux" /></p>
</td>
</tr>
<tr>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/skorp-finance">Skorp Finance</a></h4>
  <p><b>EN:</b> Personal and family finance platform (paused, 23 users, 945 transactions, 36 statement imports). Started in FastAPI and rewritten in Java 25 + Spring Boot 4 (Spring Modulith, 31 Flyway migrations, PostgreSQL 18 RLS, Testcontainers) as a practical study project to learn the Java backend stack. Includes Next.js 16 web and Expo mobile clients.<br><br><b>PT:</b> Plataforma de controle financeiro pessoal e familiar (pausada, 23 usuários, 945 transações, 36 importações de extrato). Começou em FastAPI e foi reescrita em Java 25 + Spring Boot 4 (Spring Modulith, 31 migrations Flyway, PostgreSQL 18 com RLS, Testcontainers) como projeto de estudo prático em Java. Inclui web em Next.js 16 e mobile em Expo.</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=java,spring,postgres,nextjs,ts,docker" /></p>
</td>
<td width="50%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/homelab">Homelab &amp; AI Operations Hub</a></h4>
  <p><b>EN:</b> Single-node Proxmox VE server hosting 9+ VMs/LXC containers: Coolify with Traefik reverse proxy, Cloudflare Tunnel, SeaweedFS S3, self-hosted GitHub Actions runner, and a 21-agent automation hub with 46 routine scripts, Hindsight memory, Groq/faster-whisper audio bridge, and a custom read-only Python MCP server (21 tools, 250 logged production calls).<br><br><b>PT:</b> Servidor Proxmox VE único rodando 9+ VMs/LXC: Coolify com proxy Traefik, Cloudflare Tunnel, SeaweedFS S3, runner próprio do GitHub Actions e central pessoal com 21 agentes, 46 scripts de rotina, memória Hindsight, transcrição (Groq + faster-whisper reserva) e servidor MCP próprio em Python só leitura (21 ferramentas, 250 chamadas registradas).</p>
  <p align="center"><b>Tech Stack:</b><br><img src="https://skillicons.dev/icons?i=linux,docker,python,bash,githubactions,cloudflare" /></p>
</td>
</tr>
</table>

---

### Earlier Public Tooling / Ferramenta Pública Anterior

<table width="100%">
<tr>
<td width="100%" valign="top">
  <h4 align="center"><a href="https://github.com/antonyandrade01/training-automation-suite">Training Automation Suite</a></h4>
  <p><b>EN:</b> Built during my QA role at Uniware and still in use by the team. Python CLI that queries MySQL, generates release training slide decks (<code>python-pptx</code>), and updates support tickets via the Movidesk REST API, cutting a full-day manual task down to a single command.<br><br><b>PT:</b> Feito na Uniware durante minha atuação como QA e ainda em uso pela equipe. CLI em Python que consulta MySQL, monta apresentações de treinamento de versão (<code>python-pptx</code>) e atualiza chamados via API do Movidesk, reduzindo uma tarefa manual de um dia inteiro para um comando.</p>
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
