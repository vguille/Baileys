# [Unreleased] — master (changes since v7.0.0-rc.9)

> **Contexto:** Este changelog descreve as mudanças introduzidas no branch `master`
> após a tag `v7.0.0-rc.9` (2025-11-21).  
> O objetivo é responder: **"Estava usando a RC9 sem problemas. Atualizei para o
> master e passei a ter erros constantes de desconexão com status 428
> (`connectionClosed`). O que mudou?"**
>
> 📄 **Análise detalhada com exemplos de código:** [`docs/428-regression-analysis.md`](docs/428-regression-analysis.md)  
> Inclui: diagrama de fluxo, comparação RC9 vs master, guia de diagnóstico, workarounds.

---

## 🔴 Mudanças que podem CAUSAR erros 428 no master

O código **428** (`DisconnectReason.connectionClosed`) é emitido quando o WebSocket
com os servidores do WhatsApp fecha inesperadamente — seja porque o servidor encerrou
a conexão, ou porque o cliente detectou a queda após um timeout de keep-alive.

As seções abaixo descrevem as mudanças no master que **introduziram novos
comportamentos** e que são as causas mais prováveis de 428 em instalações que
funcionavam corretamente na RC9.

---

### ⚠️ 1. Verificação obrigatória de assinatura do certificado TLS — PR #2208 (`4e681d32`)

**Risco:** 🔴 ALTO — pode impedir completamente a conexão em alguns ambientes

**O que mudou:**  
Na RC9, a validação do certificado do servidor WhatsApp estava **incompleta**:
```ts
// RC9 — validação parcial:
// TODO: handle this leaf stuff
const { issuerSerial } = proto.CertChain.NoiseCertificate.Details.decode(certIntermediate!.details!)
if (issuerSerial !== WA_CERT_DETAILS.SERIAL) {
    throw new Boom('certification match failed', { statusCode: 400 })
}
```

No master, foi adicionada validação **criptográfica completa** da cadeia de certificados:
```ts
// master — validação completa:
const verify = Curve.verify(details.key!, leaf.details, leaf.signature)
const verifyIntermediate = Curve.verify(WA_CERT_DETAILS.PUBLIC_KEY, certIntermediate.details, certIntermediate.signature)
if (!verify) throw new Boom('noise certificate signature invalid', { statusCode: 400 })
if (!verifyIntermediate) throw new Boom('noise intermediate certificate signature invalid', { statusCode: 400 })
```

A chave pública do certificado raiz está **hardcoded** em `WA_CERT_DETAILS.PUBLIC_KEY`
(`142375574d0a587166aae71ebe516437c4a28b73e3695c6ce1f7f9545da8ee6b`).

**Como se manifesta:**  
Falha no handshake com `statusCode: 400` (`'noise intermediate certificate signature invalid'`).
Como o WebSocket também fecha durante o erro, alguns clientes observam 428
(o `ws.on('close')` pode disparar depois do `end(err)` com 400).

**Mitigação:**  
Verifique se a chave pública hardcoded ainda corresponde ao certificado emitido pelo
servidor WhatsApp ao qual você está conectando. Em ambientes corporativos com proxy
TLS, esta verificação pode falhar sistematicamente.

---

### ⚠️ 2. Nova mensagem de protocolo `unified_session` após cada login — PR #2294 (`d514764`)

**Risco:** 🟡 MÉDIO — comportamento do servidor WhatsApp não documentado

**O que mudou:**  
O master envia uma nova mensagem de protocolo ao WhatsApp imediatamente após o login
bem-sucedido (`CB:success`), após o emparelhamento (`CB:iq,,pair-success`) e a cada
atualização de presença `available`:

```ts
// Enviado imediatamente após ev.emit('connection.update', { connection: 'open' })
void sendUnifiedSession()

// Também enviado em sendPresenceUpdate quando type === 'available'
if (isAvailableType) {
    void sendUnifiedSession()
}
```

O nó enviado tem formato:
```xml
<ib>
  <unified_session id="259200000" />  <!-- número gerado por fórmula de tempo -->
</ib>
```

**Como se manifesta:**  
Se o servidor WhatsApp rejeitar esta mensagem fechando o WebSocket (em vez de responder
com um erro de protocolo), o cliente recebe `ws.on('close')` → `end()` com **428**.
O erro é silencioso porque `sendUnifiedSession` captura exceções do `sendNode`:
```ts
try {
    await sendNode(node)
} catch (error) {
    logger.debug({ error }, 'failed to send unified_session telemetry')
}
```
Exceções de rede são capturadas, mas o fechamento do WebSocket pelo servidor ocorre
**fora** do try/catch.

**Diagnóstico:**  
Verifique nos logs se o `connection: 'open'` é emitido logo antes de cada 428. Se sim,
`sendUnifiedSession` é o candidato mais forte.

---

### ⚠️ 3. Nova dependência obrigatória de WASM (Rust) — PR #2315 (`b5c17411`)

**Risco:** 🔴 ALTO — falha silenciosa de inicialização quebra toda a criptografia

**O que mudou:**  
As funções `hkdf` e `md5` (usadas extensivamente na camada de criptografia, incluindo
o handshake noise) foram substituídas por implementações síncronas em WebAssembly
(Rust) via o pacote `whatsapp-rust-bridge@0.5.2`:

```ts
// master — crypto.ts
export { md5, hkdf } from 'whatsapp-rust-bridge'
```

Na RC9, `hkdf` era uma implementação JavaScript/WebCrypto assíncrona:
```ts
// RC9 — crypto.ts
export async function hkdf(...): Promise<Buffer> {
    const importedKey = await subtle.importKey(...)
    ...
}
```

**Como se manifesta:**  
- Se o módulo WASM falhar ao inicializar (incompatibilidade de Node.js, falta de suporte
  a WASM no ambiente, etc.), todas as operações de `hkdf` falham **sincronamente** com
  uma exceção não tratada.
- `processHandshake` (agora síncrono) lança o erro durante `validateConnection`,
  chamando `end(err)` com um erro interno.
- Dependendo do timing, o WebSocket pode fechar antes de `end()` ser processado,
  resultando em **428**.

**Diagnóstico:**  
```
[baileys] error in validating connection
[baileys] Error: Cannot find module 'whatsapp-rust-bridge'
```
ou
```
[baileys] Error: WebAssembly is not defined
```

**Mitigação:**  
Verifique se `whatsapp-rust-bridge` está instalado (`node_modules/whatsapp-rust-bridge`
deve existir) e se o seu ambiente suporta WebAssembly.

---

### ⚠️ 4. Mudança de comportamento padrão do `shouldSyncHistoryMessage` — commit `c81c074d`

**Risco:** 🟡 MÉDIO — quebra silenciosa para usuários com `syncFullHistory: false`

**O que mudou:**

Na RC9, a opção `syncFullHistory` controlava diretamente o comportamento de
`shouldSyncHistoryMessage` para usuários que não forneciam sua própria implementação:
```ts
// RC9 — Socket/index.ts
if (config.shouldSyncHistoryMessage === undefined) {
    newConfig.shouldSyncHistoryMessage = () => !!newConfig.syncFullHistory
    // Com syncFullHistory: false → () => false (bloqueia todo sync)
}
```

No master, o `syncFullHistory` **não afeta mais** `shouldSyncHistoryMessage`. O padrão
é agora definido globalmente em `Defaults/index.ts`:
```ts
// master — Defaults/index.ts
shouldSyncHistoryMessage: ({ syncType }) => syncType !== proto.HistorySync.HistorySyncType.FULL
// Bloqueia apenas FULL; outros tipos (PUSH_NAME, RECENT, INITIAL_BOOTSTRAP) são processados
```

**Impacto em quem usava `syncFullHistory: false`:**
- RC9: `shouldSyncHistoryMessage = () => false` → `willSyncHistory = false` → conexão
  vai para Online imediatamente.
- Master: `shouldSyncHistoryMessage = ({ syncType }) => syncType !== FULL` →
  `willSyncHistory = true` (RECENT ≠ FULL) → sistema entra em `AwaitingInitialSync`,
  **bufferiza todos os eventos** e aguarda até 20 segundos por uma notificação de sync.

Isso significa que eventos críticos de conexão (mensagens, atualizações de estado)
ficam **retidos** por até 20 segundos na primeira reconexão. Em loops de reconexão
(por outras causas), isso pode fazer parecer que a conexão está instável ou lenta.

**Mitigação:**  
Se você usa `syncFullHistory: false` e quer o comportamento da RC9, defina
explicitamente:
```ts
makeWASocket({
    syncFullHistory: false,
    shouldSyncHistoryMessage: () => false,
    // ...
})
```

---

### ⚠️ 5. Reescrita completa do noise handler — PR #2182 (`5887551d`) + PR #2284 (`ffc019fb`)

**Risco:** 🟡 MÉDIO — reescrita de componente crítico, concorrência potencial no `processData`

**O que mudou:**  
O handler de ruído (camada de criptografia do WebSocket) foi completamente reescrito
para corrigir uma race condition entre os contadores de leitura/escrita durante o
handshake. A nova implementação usa uma classe `TransportState` e uma função
`processData` centralizada.

**Nova race condition potencial introduzida:**  
Enquanto `processData` aguarda `await decodeBinaryNode(result)` (que é assíncrono
porque pode descomprimir frames zlib), uma nova mensagem pode chegar pelo WebSocket e
chamar `decodeFrame` novamente, iniciando uma **segunda instância concorrente de
`processData`** sobre o mesmo buffer `inBytes`. Embora o modelo single-threaded do
Node.js previna corrupção de dados entre pontos de `await`, a ordem de processamento
de frames pode ser alterada em condições de alta carga.

**Como se manifesta:**  
- Frames processados fora de ordem → falha de autenticação MAC → servidor fecha a
  conexão → **428**.
- Pode ocorrer intermitentemente, especialmente durante o sync inicial (muitos frames
  chegando ao mesmo tempo).

---

### ⚠️ 6. `saveIdentity` chamado antes de cada descriptografia `pkmsg` — PR #2307 (`b023901`)

**Risco:** 🟡 MÉDIO — risco de loop de invalidação de sessão

**O que mudou:**  
Para detectar mudanças de chave de identidade (contato reinstalou o WhatsApp), o
master agora chama `storage.saveIdentity()` **antes** de cada descriptografia de
mensagem do tipo `pkmsg` (PreKeyWhisperMessage):

```ts
// master — libsignal.ts
if (type === 'pkmsg') {
    const identityKey = extractIdentityFromPkmsg(ciphertext)
    if (identityKey) {
        const identityChanged = await storage.saveIdentity(addrStr, identityKey)
        // Se mudou → sessão é limpa atomicamente
    }
}
```

**Risco de loop:**  
Se `extractIdentityFromPkmsg` retornar um resultado inconsistente (por exemplo, em
mensagens com formato inesperado ou versão de protocolo diferente), `saveIdentity`
pode sinalizar uma "mudança de identidade" incorretamente, limpando a sessão. Isso
geraria uma nova rodada de `pkmsg` → novo `saveIdentity` → nova limpeza → loop.

Cada iteração do loop pode forçar re-registros de chave que sobrecarregam o servidor,
que eventualmente encerra a conexão com **428**.

---

### ⚠️ 7. Redesign dos mutexes — PR #2137 (`829fa8d6`)

**Risco:** 🟡 MÉDIO — mudança de comportamento de ordenação de eventos

**O que mudou:**  
A RC9 usava um único mutex (`processingMutex`) para serializar **todos** os
processamentos (mensagens, recibos, notificações, patches de estado). O master
introduziu **4 mutexes separados** (`messageMutex`, `receiptMutex`, `notificationMutex`,
`appStatePatchMutex`).

**Impacto:**  
Os 4 tipos de eventos agora são processados **de forma independente e concorrente entre
si**. Na RC9, a garantia de ordenação global impedia que, por exemplo, um recibo de leitura
fosse processado antes da mensagem correspondente chegar. No master, isso deixou de ser
garantido entre tipos diferentes de mutex.

Em cenários com muitos eventos simultâneos, esta mudança pode levar a estados
inconsistentes no app que, em casos extremos, resultam em reconexões forçadas.

---

### ⚠️ 8. `end()` virou `async` mas `mapWebSocketError` não faz `await` — commit `282f065`

**Risco:** 🟡 MÉDIO — exceções dentro de `end()` podem ficar sem handler

**O que mudou:**
```ts
// RC9:
const end = (error: Error | undefined) => { ws.close() }
ws.on('error', mapWebSocketError(end))  // end: (err) => void ✓

// master:
const end = async (error: Error | undefined) => { await ws.close() }
ws.on('error', mapWebSocketError(end))  // mapWebSocketError ainda espera (err) => void
```

`mapWebSocketError` chama `end(...)` mas não faz `await`. Qualquer exceção que
`ws.close()` lance depois do `await` fica como `UnhandledPromiseRejection`.

---

### ⚠️ 9. `presenceSubscribe` agora faz DB read antes de enviar — PR #2257 (`349e7bd`)

**Risco:** 🟡 BAIXO-MÉDIO — introduz janela de tempo onde o WS pode fechar entre a leitura e o envio

**O que mudou:**
```ts
// RC9 — síncrono, envia imediatamente:
const presenceSubscribe = (toJid: string, tcToken?: Buffer) => sendNode(...)

// master — assíncrono, lê store primeiro:
const presenceSubscribe = async (toJid: string) => {
    const tcTokenContent = await buildTcTokenFromJid({ authState, jid: toJid })
    return sendNode(...)  // ← WS pode ter fechado durante o await acima
}
```

O mesmo padrão foi aplicado a `profilePictureUrl`. Entre o `await buildTcTokenFromJid`
e o `sendNode`, o WebSocket pode ter sido fechado (por `sendUnifiedSession` ou outra causa).
O `sendNode` lança 428 nesse caso.

---

## ✅ Fixes incluídos no master (que NÃO causam 428)

Os commits abaixo corrigem bugs reais que existiam na RC9, mas **não são a causa** dos
erros 428 em quem estava estável na RC9:

| Commit | PR | Resumo |
|--------|-----|--------|
| `5cbad31` | #2264 | Corrige "Online mas desconectado" — tratamento correto de notificações de mudança de identidade |
| `282f065` | — | `end()` agora é `async`, evita fechamento duplicado do socket |
| `1408499` | #2167 | Evita recriação prematura de sessão (`retryCount < 2`) |
| `a2677c8` | #2160 | Corrige memory leak crítico no event buffer |
| `1e6f65c` | #2151 | Corrige memory leak no `makeMutex` |
| `92d4198` | #2280 | Ignora retry de status messages expiradas (+24h) |
| `56ee829` | — | Remove timeout de 500ms antes de emitir eventos PDO |
| `a7a53ad` | #2153 | Libera mutex de transação corretamente |
| `674f116` | #2147 | Cancela requisições PDO pendentes ao iniciar retry |

---

## ✨ Novas funcionalidades

| Commit | PR | Descrição |
|--------|-----|-----------|
| `d514764` | #2294 | `sendUnifiedSession` — sessão unificada após login |
| `4e681d32` | #2208 | Verificação completa da assinatura do certificado TLS |
| `b5c17411` | #2315 | Criptografia síncrona WASM (Rust) — `hkdf` e `md5` |
| `349e7bd` | #2257 | TCToken em atualizações de perfil e presença |
| `23156c8` | #2190 | Número de telefone do chamador no evento `call` |
| `925ed6a` | #2057 | `settings.update` para configurações de appstate |
| `2504774` | #2198 | LabelMember — rótulos de membros em grupos |
| `c392d4c` | #2189 | Suporte a JIDs FB/Interop e strings vazias |
| `c9c3481` | #1906 | Tokens para denúncia de mensagens |
| `b023901` | #2307 | Detecção de mudança de chave de identidade + MAC errors |
| `7a5b090` | #2334 | Resend de placeholder para mensagens CTWA sem criptografia |

---

## ⚡ Melhorias de desempenho

| Commit | PR | Descrição |
|--------|-----|-----------|
| `829fa8d` | #2137 | Redesign do mutex (4 mutexes separados) |
| `b7960db` | #2138 | Processamento de nodes offline em lotes com yield |
| `fa2a837` | #2316 | Redução de chamadas ao DB durante sync |
| `90e8ba8` | #2093 | Cache de filhos de `BinaryNode` |
| `4609a37` | #2179 | Preservação de tipo no event buffer |

---

## 🐛 Outras correções

- **PR #2282** (`f829b6d`): Extração de mapeamentos LID-PN de objetos de conversa.
- **PR #2274** (`52fcad2`): Otimização de `getLIDsForPNs` e adição de `getPNsForLIDs`.
- **PR #2268** (`a89736f`): Extração de mapeamentos LID-PN de `phoneNumberToLidMappings`.
- **PR #2266** (`8ff01b8`): Armazenamento de mapeamentos LID-PN do `contactAction` sync.
- **PR #2277** (`32134a8`): `messageTimestamp` nos updates de status de mensagem.
- **PR #2258** (`d36d9c1`): Verificações de `groupStatusMessage` no processamento.
- **PR #2226** (`432c26a`): Falhas de criptografia por destinatário tratadas individualmente.
- **PR #2183** (`720cc68`): Preservação de campos vazios em perfis de negócios.
- **PR #2180** (`0b3b2a8`): Verificações de nulidade em geração de conteúdo de mensagens.
- **PR #2136** (`96e4e04`): `await` adicionado nas verificações de cache para resend.
- **PR #1991** (`9611a1a`): Tratamento de `string` em campos `long` no WAProto JSON.
- **PR #1874** (`a1d69f7`): `associatedChildMessage` em `normalizeMessageContent`.
- **PR #2245** (`81d9c12`): `getMessageType` alinhado com WA Web.
- **PR #2130** (`d4ef73a`): Workflow CI de atualização automática de versão WA.
- **PR #2128** (`43d1787`): Melhoria no upload de mídia e redução de pressão de memória.

---

## 📦 Dependências e infra

- **Nova** dependência `whatsapp-rust-bridge@0.5.2` (WASM síncrono para `hkdf`/`md5`).
- Atualização do certificado raiz WA: emissor `WhatsAppLongTerm1`, chave pública
  definida em `WA_CERT_DETAILS.PUBLIC_KEY`.
- Versão do WhatsApp Web: `[2, 3000, 1027934701]` → `[2, 3000, 1033846690]`.
- Workflow CI para atualização automática de versão WA adicionado.

---

## ⚠️ Breaking changes / mudanças de comportamento

| Área | Comportamento RC9 | Comportamento master |
|------|-------------------|----------------------|
| `shouldSyncHistoryMessage` padrão | `() => !!syncFullHistory` (controlado pela flag) | `syncType !== FULL` sempre — `syncFullHistory: false` é **ignorado** para este propósito |
| `noise.processHandshake` | `async` (retorna Promise) | **síncrono** — remover `await` ao chamar |
| `MessageRetryManager.shouldRecreateSession` | `(jid, retryCount, hasSession)` | `(jid, hasSession, errorCode?)` — `retryCount` removido da assinatura |
| `end(error)` interno do socket | função síncrona | função `async` — todos os call sites usam `void end(...)` |
| `ws.close()` | síncrono | `async` — aguarda o evento `close` do WebSocket |
| Processamento de eventos | mutex único global | 4 mutexes independentes (mensagens, recibos, notificações, patches) |

---

## 🔍 Guia de diagnóstico para erros 428 no master

Se você está vendo erros 428 frequentes no master e a RC9 funcionava:

1. **Verifique os logs imediatamente antes do 428:**
   - `'connected to WA'` aparece seguido de 428 em segundos? → Problema no handshake
     ou em `sendUnifiedSession`.
   - `'connection: open'` é emitido antes do 428? → `sendUnifiedSession` é o candidato
     mais provável.

2. **Verifique a inicialização do WASM:**
   ```
   grep -i "wasm\|rust-bridge\|Cannot find module" <arquivo-de-log>
   ```

3. **Teste desabilitando `sendUnifiedSession` temporariamente:**  
   Não há opção de configuração para isso ainda — seria necessário patching manual.

4. **Se usava `syncFullHistory: false`, adicione explicitamente:**
   ```ts
   shouldSyncHistoryMessage: () => false
   ```

5. **Verifique a versão do Node.js** — o WASM requer Node.js ≥ 16 com suporte a
   `WebAssembly.instantiateStreaming`.

---

# 7.0.0-rc.9 (2025-11-21)


