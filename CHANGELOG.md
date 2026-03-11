# [Unreleased] — master (changes since v7.0.0-rc.9)

> **Nota / Note:** Este changelog descreve todas as mudanças entre a tag `v7.0.0-rc.9`
> (2025-11-21) e o branch `master` atual. Usuários que migram do RC9 para o master devem
> ler com atenção a seção sobre **erros 428 / connectionClosed**.
>
> This changelog describes all changes between the `v7.0.0-rc.9` tag (2025-11-21) and
> the current `master` branch. Users migrating from RC9 to master should pay attention
> to the section on **428 / connectionClosed errors**.

---

## 🔴 Fixes relacionados a desconexões com status 428 (`connectionClosed`)

O código de status **428** (`DisconnectReason.connectionClosed`) é emitido sempre que
a conexão WebSocket com os servidores do WhatsApp é encerrada inesperadamente. Nas
versões anteriores ao RC9 e no período entre RC9 e o master atual, diversas causas
independentes contribuíam para desconexões frequentes. As correções abaixo eliminam
as principais:

### 1. Race condition no handler de ruído (noise handler) — #2182
**Commit:** `5887551`  
**Impacto:** **CRÍTICO** — causava falhas de descriptografia que resultavam em
`connectionClosed` (428) repetitivos.

O `decodeFrame` processava múltiplos frames de forma concorrente. Quando dois frames
chegavam ao mesmo tempo durante o handshake ou logo após, os contadores de leitura
(`readCounter`) e escrita (`writeCounter`) ficavam dessincronizados, gerando erros de
MAC (autenticação de mensagem). O WhatsApp encerra a sessão ao detectar um MAC inválido.

**Solução:** A classe `TransportState` encapsula contadores independentes de leitura e
escrita. O mecanismo `pendingOnFrame` garante que os frames chegados durante a transição
de estado sejam processados na ordem correta.

### 2. Conexão mostrando "Online" mas efetivamente desconectada — #2132 / #2264
**Commit:** `5cbad31`  
**Impacto:** **ALTO** — o socket ficava preso em estado inconsistente sem emitir
`connectionClosed`.

Notificações de mudança de identidade (`identity`) não eram tratadas corretamente,
fazendo com que a conexão parecesse ativa mas não conseguisse descriptografar
mensagens. Adicionado o handler `handleIdentityChange` com debounce e validação de
sessão por JID e LID.

### 3. Condição de corrida na finalização da conexão (`end`) — commit `282f065`
**Impacto:** **MÉDIO** — causava erros silenciosos ao fechar a conexão, podendo gerar
múltiplos eventos `connectionClosed`.

A função `end` era síncrona: `ws.close()` não era aguardado (`await`). Em situações de
alta latência, o WebSocket podia emitir um evento `close` antes de a limpeza de
listeners ser concluída, disparando `end` mais de uma vez. Agora `end` é `async` e
`ws.close()` é aguardado corretamente.

### 4. Sessão sendo recriada cedo demais — #2167
**Commit:** `1408499`  
**Impacto:** **MÉDIO** — recriação prematura de sessão causava instabilidade e
potencialmente mais desconexões.

`shouldRecreateSession` era chamado com `retryCount = 1`, fazendo com que sessões
fossem descartadas na *primeira* falha de descriptografia. A correção garante que a
lógica só é ativada com `retryCount > 1`. Além disso, o parâmetro `retryCount` foi
removido da assinatura do método — agora a decisão é baseada em histórico de
recriações por JID e no tipo de erro (MAC errors).

### 5. Detecção de mudança de chave de identidade — #2307
**Commit:** `b023901`  
**Impacto:** **ALTO** — sem este fix, mensagens de contatos que reinstalaram o
WhatsApp não podiam ser descriptografadas, gerando retries infinitos.

Alinhado com o comportamento do WhatsApp Web: ao receber um `pkmsg` (PreKeyWhisperMessage),
a chave de identidade do remetente é extraída e comparada com a armazenada. Se mudou,
a sessão é limpa atomicamente *antes* da tentativa de descriptografia.

### 6. MAC errors agora forçam recriação imediata de sessão — #2307
**Commit:** `b023901`  
**Impacto:** **ALTO** — evita loops de retry para sessões definitivamente
fora de sincronia.

Adicionado o enum `RetryReason` com os códigos de erro do Signal usados pelo WhatsApp
Web. Erros de MAC (`SignalErrorInvalidMessage = 4` e `SignalErrorBadMac = 7`) agora
disparam recriação imediata de sessão em vez de aguardar o timeout normal.

### 7. Memory leak crítico no event buffer — #2160
**Commit:** `a2677c8`  
**Impacto:** **MÉDIO** — o vazamento de memória degradava o processo ao longo do
tempo, causando timeouts e quedas de conexão.

Promises não eram coletadas pelo garbage collector no `makeMutex`. Corrigido retendo
referências fracas e limpando o mapa de promises após resolução.

### 8. Memory leak no makeMutex — #2151
**Commit:** `1e6f65c`  
**Impacto:** **MÉDIO** — mesmo root cause do item anterior, na implementação do mutex.

### 9. `shouldSyncHistoryMessage` desabilitado bloqueia mapeamentos LID — commit `1ef04d5`
**Impacto:** **ALTO** — se todos os tipos de sync forem bloqueados por
`shouldSyncHistoryMessage`, o Baileys não consegue construir os mapeamentos
LID ↔ número de telefone, causando falhas de criptografia em grupos e
potencialmente desconexões.

Adicionado aviso explícito (`logger.warn`) caso toda a sincronização seja desativada.
O valor padrão de `shouldSyncHistoryMessage` foi alterado para permitir todos os tipos
**exceto** `FULL` (sync completo de histórico), reduzindo o consumo de memória sem
comprometer os mapeamentos essenciais.

---

## ✨ Novas funcionalidades

### `sendUnifiedSession` — PR #2294 (`d514764`)
Envia um identificador de sessão unificado ao WhatsApp após o login e ao marcar
presença como `available`. Alinha o comportamento do Baileys com o WhatsApp Web,
potencialmente reduzindo desconexões iniciadas pelo servidor.

### Verificação de assinatura folha (leaf signature) — PR #2208 (`4e681d3`)
Valida a assinatura da chave de identidade do dispositivo durante o handshake,
aumentando a segurança do emparelhamento.

### Criptografia síncrona com WASM (Rust) para app state sync — PR #2315 (`b5c1741`)
Substituição das operações assíncronas de `hkdf` e `md5` por implementações síncronas
em WebAssembly (Rust). Elimina pontos de concorrência que podiam gerar condições de
corrida durante o processamento do estado do app.

### Token TCToken em atualizações de perfil e presença — PR #2257 (`349e7bd`)
Enviado junto com atualizações de perfil e subscribe de presença, alinhando com o
protocolo atual do WhatsApp Web.

### Número de telefone do chamador no evento de chamada recebida — PR #2190 (`23156c8`)
O evento `call` agora inclui o campo `callerPhoneNumber` (quando disponível).

### Emissão de eventos de configurações (`appstate`) — PR #2057 (`925ed6a`)
Configurações como `defaultDisappearingMode` e outras agora emitem eventos
`settings.update`.

### LabelMember — PR #2198 (`2504774`)
Suporte a rótulos de membros em grupos.

### Suporte a JIDs FB/Interop e strings vazias — PR #2189 (`c392d4c`)
Codificação/decodificação de JIDs de Facebook/Interoperabilidade.

### Relatório de mensagens (reporting tokens) — PR #1906 (`c9c3481`)
Suporte ao fluxo de denúncia de mensagens via tokens.

---

## ⚡ Melhorias de desempenho

### Redesign do mutex — PR #2137 (`829fa8d`)
Mutexes separados para processamento de mensagens, recibos, app state patches e
notificações. Melhora o throughput e reduz a probabilidade de deadlocks.

### Otimização do processamento de nodes offline — PR #2138 (`b7960db`)
Processamento em lote com yield do event loop para evitar bloqueio do thread
principal durante sincronização.

### Redução de chamadas ao banco de dados durante sincronização — PR #2316 (`fa2a837`)
Cache e batching de operações de DB durante o sync inicial reduzem a latência de
conexão.

### Cache de filhos de `BinaryNode` — PR #2093 (`90e8ba8`)
`getBinaryNodeChild/ren` agora usa cache para evitar travessia repetida de arrays.

### Prevenção de perda de tipo no event buffer — PR #2179 (`4609a37`)
Tipos de eventos agora são preservados corretamente durante o buffering.

---

## 🐛 Outras correções

- **PR #2334** (`7a5b090`): Resend de placeholder para mensagens sem criptografia
  (anúncios CTWA). Respeita limite de 14 dias do WA Web.
- **PR #2282** (`f829b6d`): Extração de mapeamentos LID-PN de objetos de conversa
  durante history sync.
- **PR #2274** (`52fcad2`): Otimização de `getLIDsForPNs` e adição de `getPNsForLIDs`.
- **PR #2268** (`a89736f`): Extração de mapeamentos LID-PN de
  `phoneNumberToLidMappings` no history sync.
- **PR #2266** (`8ff01b8`): Armazenamento de mapeamentos LID-PN do `contactAction` sync.
- **PR #2280** (`92d4198`): Skip de retry para mensagens de status com mais de 24h.
- **PR #2277** (`32134a8`): `messageTimestamp` incluído nos updates de status de
  mensagem em `messages-recv`.
- **PR #2258** (`d36d9c1`): Verificações de `groupStatusMessage` no processamento de
  mensagens.
- **PR #2226** (`432c26a`): Falhas de criptografia por destinatário agora são tratadas
  individualmente; a operação falha apenas quando *todos* os destinatários falham.
- **PR #2183** (`720cc68`): Preservação de campos vazios em perfis de negócios.
- **PR #2180** (`0b3b2a8`): Verificações de nulidade aprimoradas na geração de
  conteúdo de mensagens.
- **PR #2147** (`674f116`): Cancelamento de requisições telefônicas pendentes ao
  iniciar retry de mensagem.
- **PR #2136** (`96e4e04`): `await` adicionado nas verificações de cache para lógica
  de resend.
- **commit** `56ee829`: Remoção do timeout antes de emitir eventos em
  `process-message`.
- **commit** `a7a53ad` (PR #2153): Liberação do mutex de transação quando as
  referências chegam a zero.
- **PR #1991** (`9611a1a`): Tratamento de valores `string` em campos `long` durante
  serialização JSON do WAProto.
- **PR #1874** (`a1d69f7`): `associatedChildMessage` adicionado como opção em
  `normalizeMessageContent`.
- **PR #2245** (`81d9c12`): `getMessageType` alinhado com o comportamento do WhatsApp.
- **PR #2130** (`d4ef73a`): Workflow automatizado de atualização de versão do WhatsApp Web.
- **PR #2128** (`43d1787`): Melhoria no upload de mídia e redução de pressão de
  memória.

---

## 📦 Dependências e infra

- Adicionada dependência `whatsapp-rust-bridge@0.5.2` (criptografia WASM síncrona).
- Atualização do certificado raiz do WA: serial `0`, emissor `WhatsAppLongTerm1`,
  chave pública atualizada em `WA_CERT_DETAILS`.
- Versão do WhatsApp Web atualizada de `[2, 3000, 1027934701]` para
  `[2, 3000, 1033846690]`.
- Workflow CI adicionado para atualização automática de versão.

---

## ⚠️ Breaking changes / mudanças de comportamento

| Área | RC9 | master |
|------|-----|--------|
| `shouldSyncHistoryMessage` padrão | `() => true` (sync completo) | Bloqueia apenas `HistorySyncType.FULL`; todos os outros tipos são sincronizados |
| `noise.processHandshake` | `async` | **síncrono** — remover `await` ao chamar |
| `MessageRetryManager.shouldRecreateSession` | recebe `(jid, retryCount, hasSession)` | recebe `(jid, hasSession, errorCode?)` — `retryCount` removido |
| `end(error)` interno do socket | síncrono | `async` — chamadas sem `await` usam `void end(...)` |

---

# 7.0.0-rc.9 (2025-11-21)


