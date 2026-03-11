# Análise de Regressão — Erros 428 no master vs v7.0.0-rc.9

> **Cenário:** Estava estável na RC9. Após atualizar para o master atual, erros
> `428 (connectionClosed)` passaram a ocorrer **durante ações**: enviar mensagens,
> obter dados de grupos, assinar presença, entre outros.

---

## O que é o erro 428?

```
DisconnectReason.connectionClosed = 428
```

No Baileys, o código 428 é emitido em dois cenários distintos:

| Cenário | Origem do código |
|---------|-----------------|
| **A — O WebSocket com o WA fecha** | `ws.on('close', ...)` chama `end(new Boom(..., { statusCode: 428 }))` |
| **B — Uma ação tenta usar uma conexão já fechada** | `sendRawMessage` ou `awaitNextMessage` verificam `ws.isOpen` e lançam `new Boom('Connection Closed', { statusCode: 428 })` |

Os erros que ocorrem **durante ações** (enviar mensagem, obter dados de grupos, etc.)
pertencem ao cenário **B** — você chama uma função como `sendMessage` ou
`groupMetadata` e ela retorna `Error: Connection Closed (428)` porque o WebSocket já
estava fechado no momento da chamada (ou fechou durante a espera pela resposta).

Portanto, **o verdadeiro problema é o que causou o fechamento do WebSocket** — e há
várias mudanças no master que fazem isso acontecer com mais frequência.

---

## Diagrama: fluxo de uma ação até o 428

```
Usuário chama: sendMessage() / groupMetadata() / presenceSubscribe()
    │
    ▼
relayMessage() / query() / sendNode()
    │
    ▼
sendRawMessage()
    ├── ws.isOpen == false? → lança Boom 428 imediatamente (Cenário B direto)
    └── ws.isOpen == true  → envia frame
                               │
                               ▼
                          waitForMessage() aguarda resposta do servidor
                               │
                               ├── ws fecha antes da resposta? → ws.on('close') rejeita a Promise → Boom 428
                               └── resposta chega → OK
```

---

## Mudanças no master que CAUSAM mais fechamentos de WebSocket

### 1. `sendUnifiedSession` após cada login e cada `available` — PR #2294 (`d514764`)

**Arquivo:** `src/Socket/socket.ts` e `src/Socket/chats.ts`

**Antes (RC9):** A mensagem `<ib><unified_session/>` não existia.

**Depois (master):**
```ts
// Em CB:success (login bem-sucedido):
ev.emit('connection.update', { connection: 'open' })
void sendUnifiedSession()   // ← NOVA linha

// Em sendPresenceUpdate quando type === 'available':
if (isAvailableType) {
    void sendUnifiedSession()  // ← NOVA linha
}
```

**Por que causa 428 durante ações:**  
`sendUnifiedSession` envia `<ib><unified_session id="..."/>` ao servidor. Se o
servidor WhatsApp fechar o WebSocket em resposta a essa mensagem (comportamento não
documentado), o socket cai. Como o `sendUnifiedSession` usa `void` (sem `await`), a
exception que o `sendNode` lançaria ao detectar a queda **nunca é capturada**. O
WebSocket fecha silenciosamente. A próxima ação do usuário (`sendMessage`,
`groupMetadata`) tenta usar o WebSocket já fechado e recebe 428 imediatamente.

```ts
const sendUnifiedSession = async () => {
    if (!ws.isOpen) return

    try {
        await sendNode(node)  // se o WS fecha DEPOIS de sendNode, ws.on('close') é disparado
    } catch (error) {
        logger.debug({ error }, 'failed to send unified_session telemetry')
        // Apenas exceções SÍNCRONAS de sendNode são capturadas aqui
        // O fechamento assíncrono do WS pelo servidor não é capturado
    }
}
```

---

### 2. `presenceSubscribe` agora é assíncrona e faz nova requisição de DB — PR #2257 (`349e7bd`)

**Arquivo:** `src/Socket/chats.ts`

**Antes (RC9):**
```ts
const presenceSubscribe = (toJid: string, tcToken?: Buffer) =>
    sendNode({ tag: 'presence', attrs: { to: toJid, type: 'subscribe' }, content: tcToken ? [...] : undefined })
```

**Depois (master):**
```ts
const presenceSubscribe = async (toJid: string) => {
    const tcTokenContent = await buildTcTokenFromJid({ authState, jid: toJid })
    //                     ^^^^^^^^ Lê 'tctoken' da store ANTES de enviar
    return sendNode({ ..., content: tcTokenContent })
}
```

**Risco:** Entre `await buildTcTokenFromJid` e o `sendNode`, o WebSocket pode ter
fechado por outra razão (por ex., o `sendUnifiedSession` descrito acima). O `sendNode`
lançará 428.

A mesma mudança foi aplicada a `profilePictureUrl` — agora também lê `tctoken` da
store antes de cada consulta.

---

### 3. `end()` tornou-se `async` mas `mapWebSocketError` não foi atualizado — commits `282f065`, `b1c76ebe`

**Arquivo:** `src/Socket/socket.ts`

**Antes (RC9):**
```ts
const end = (error: Error | undefined) => { ... }
ws.on('error', mapWebSocketError(end))  // end é síncrono ✓
```

**Depois (master):**
```ts
const end = async (error: Error | undefined) => { ... }  // ← agora é async
ws.on('error', mapWebSocketError(end))  // ← mapWebSocketError espera (err: Error) => void
```

```ts
function mapWebSocketError(handler: (err: Error) => void) {  // ← tipo não atualizado
    return (error: Error) => {
        handler(...)  // chama end() mas não faz await
        // se end() falhar, a Promise rejeitada não é tratada
    }
}
```

**Risco:** Se `ws.close()` dentro de `end()` lançar uma exceção (o que pode acontecer
quando o WebSocket já está em estado inconsistente), a Promise rejeitada fica
"flutuando" sem handler (`UnhandledPromiseRejection`). Dependendo da versão do Node.js,
isso pode encerrar o processo ou deixar recursos não liberados.

---

### 4. `makeMutex` reimplementado com `async-mutex` — commit `829fa8d` + `1e6f65c`

**Arquivo:** `src/Utils/make-mutex.ts`

**Antes (RC9):** Implementação simples de Promise-chain:
```ts
export const makeMutex = () => {
    let task = Promise.resolve() as Promise<any>
    return {
        mutex<T>(code: () => Promise<T> | T): Promise<T> {
            task = (async () => {
                try { await task } catch {}
                return code()  // erros na task anterior são ENGOLIDOS, fila nunca trava
            })()
            return task
        }
    }
}
```

**Depois (master):** Usa `async-mutex`:
```ts
import { Mutex as AsyncMutex } from 'async-mutex'
export const makeMutex = () => {
    const mutex = new AsyncMutex()
    return {
        mutex<T>(code: () => Promise<T> | T): Promise<T> {
            return mutex.runExclusive(code)
        }
    }
}
```

**Diferença crítica de comportamento:**  
Na RC9, erros em `code()` eram propagados ao chamador mas a **fila nunca travava** —
a próxima `mutex()` era executada independentemente. Com `async-mutex`, se `code()`
lançar um erro e o chamador não tratar corretamente, o comportamento depende da
implementação interna do `async-mutex`. Em situações onde a conexão cai no meio do
processamento (e.g., `messageMutex.mutex(async () => { await decrypt(); ... })`), a
fila pode ficar em estado inconsistente por mais tempo.

**Mudança adicional em `makeKeyedMutex`:**  
O `makeKeyedMutex` (usado em `relayMessage` para serializar criptografia por JID) foi
reescrito com reference counting. Se `entry.refCount` ficar negativo por algum bug de
contagem, o mutex para aquele JID é deletado e as próximas chamadas criam um novo
mutex, quebrando a garantia de serialização.

---

### 5. 4 mutexes separados em vez de 1 — PR #2137 (`829fa8d`)

**Arquivo:** `src/Socket/chats.ts` e `src/Socket/messages-recv.ts`

**Antes (RC9):** Um único `processingMutex` serializa **tudo**:
```ts
const processingMutex = makeMutex()
// Usado em: handleReceipt, handleNotification, handleMessage, appPatch
```

**Depois (master):** 4 mutexes independentes:
```ts
const messageMutex = makeMutex()
const receiptMutex = makeMutex()
const appStatePatchMutex = makeMutex()
const notificationMutex = makeMutex()
```

**Impacto no comportamento:**  
Com `processingMutex` global, a sequência:
```
[receipt: messageA delivered] → [message: messageB arrives] → [notification: group update]
```
era estritamente ordenada. Agora, notificações e recibos são processados em paralelo
com mensagens. Isso pode levar a:
- Uma mensagem `messageB` sendo processada ANTES do recibo de `messageA` ser committed.
- Uma notificação de mudança de grupo sendo processada enquanto uma mensagem do grupo
  ainda está sendo descriptografada.
- Estado inconsistente no app que faz chamadas subsequentes esperarem por dados que
  ainda não chegaram, aumentando timeouts.

---

### 6. `saveIdentity` antes de cada decriptação `pkmsg` — PR #2307 (`b023901`)

**Arquivo:** `src/Signal/libsignal.ts`

**Antes (RC9):** `isTrustedIdentity` retornava sempre `true` sem verificar nada.

**Depois (master):**
```ts
if (type === 'pkmsg') {
    const identityKey = extractIdentityFromPkmsg(ciphertext)
    if (identityKey) {
        const addrStr = addr.toString()
        const identityChanged = await storage.saveIdentity(addrStr, identityKey)
        // saveIdentity faz uma transação na store para cada mensagem pkmsg recebida
    }
}
```

**Impacto em `sendMessage`:**  
`saveIdentity` abre uma transação (`keys.transaction`) na store de autenticação.
Transações são serializadas por chave via `getTxMutex`. Se você estiver enviando uma
mensagem para um contato que acabou de reinstalar o WhatsApp (gerando `pkmsg`), a
transação de `saveIdentity` concorre com a transação de `encryptMessage`. Em
implementações de store com latência (ex: SQLite em disco), isso pode causar timeouts
ou travamentos de lock que resultam em erros.

---

### 7. `retryCount > 1` antes de `shouldRecreateSession` — commit `1408499`

**Arquivo:** `src/Socket/messages-recv.ts`

**Antes (RC9):**
```ts
if (enableAutoSessionRecreation && messageRetryManager) {
    // Chamado desde o PRIMEIRO retry (retryCount = 1)
    const result = messageRetryManager.shouldRecreateSession(fromJid, retryCount, hasSession.exists)
}
```

**Depois (master):**
```ts
if (enableAutoSessionRecreation && messageRetryManager && retryCount > 1) {
    //                                                     ^^^^^^^^^ nova guarda
    const result = messageRetryManager.shouldRecreateSession(fromJid, hasSession.exists)
}
```

**Impacto:** Na primeira falha de descriptografia, a sessão **não** é mais recriada.
Isso evita recriação prematura, mas significa que o cliente enviará um `retry receipt`
(NACK) ao servidor sem recriar a sessão. Se o servidor interpretar isso como sinal de
que a sessão está corrompida, pode encerrar a conexão — resultando em 428.

---

### 8. Verificação obrigatória de assinatura do certificado TLS — PR #2208 (`4e681d32`)

**Arquivo:** `src/Utils/noise-handler.ts`

**Antes (RC9):**
```ts
// TODO: handle this leaf stuff
const { issuerSerial } = proto.CertChain.NoiseCertificate.Details.decode(certIntermediate!.details!)
if (issuerSerial !== WA_CERT_DETAILS.SERIAL) {
    throw new Boom('certification match failed', { statusCode: 400 })
}
// Sem verificação criptográfica real
```

**Depois (master):**
```ts
const verify = Curve.verify(details.key!, leaf.details, leaf.signature)
const verifyIntermediate = Curve.verify(
    WA_CERT_DETAILS.PUBLIC_KEY,  // chave hardcoded: 142375574d0a587166...
    certIntermediate.details,
    certIntermediate.signature
)
if (!verify) throw new Boom('noise certificate signature invalid', { statusCode: 400 })
if (!verifyIntermediate) throw new Boom('noise intermediate certificate signature invalid', { statusCode: 400 })
```

**Impacto:** Qualquer divergência entre a chave hardcoded e o certificado real do
servidor causa falha no handshake. O WebSocket fecha com `statusCode: 400`, que por
vezes é observado pelo caller como 428 (o `ws.on('close')` dispara após o `end(err)`
com 400).

---

### 9. Nova dependência WASM obrigatória — PR #2315 (`b5c17411`)

**Arquivo:** `src/Utils/crypto.ts`

**Antes (RC9):**
```ts
export async function hkdf(...): Promise<Buffer> {
    const importedKey = await subtle.importKey(...)  // WebCrypto nativo
    ...
}
```

**Depois (master):**
```ts
export { md5, hkdf } from 'whatsapp-rust-bridge'  // Rust WASM síncrono
```

**`processHandshake` tornou-se síncrono:**
```ts
// RC9:
const keyEnc = await noise.processHandshake(handshake, creds.noiseKey)
// master:
const keyEnc = noise.processHandshake(handshake, creds.noiseKey)  // síncrono
```

**Impacto:** Se o WASM não inicializar corretamente, `hkdf` lança uma exceção
síncrona durante o handshake, fazendo `validateConnection` falhar e chamar
`end(err)`. O WebSocket fecha e não há reconexão automática.

---

### 10. Event buffer: `bufferCount` resetado a 0 em vez de incrementado — commit `a2677c8`

**Arquivo:** `src/Utils/event-buffer.ts`

**Antes (RC9):**
```ts
if (!isBuffering) {
    isBuffering = true
    bufferCount++
} else {
    bufferCount++
}
```

**Depois (master):**
```ts
if (!isBuffering) {
    isBuffering = true
    bufferCount = 0  // ← RESETA ao invés de incrementar
}
bufferCount++  // incrementa depois
```

**Impacto sutil:** Se `buffer()` for chamado novamente enquanto `isBuffering` já é
`true` (durante reconexão), o count não é incrementado na branch `!isBuffering`. O
resultado líquido é o mesmo (ambos terminam com `bufferCount + 1`), mas o **reset a 0**
dentro do `if (!isBuffering)` significa que, na PRIMEIRA chamada a `buffer()` após
uma reconexão, `bufferCount` vai de `N` para `0+1 = 1` em vez de `N+1`. Isso pode
fazer com que um `flush()` de outro contexto veja `bufferCount === 0` prematuramente
e flush os eventos antes de toda a inicialização estar completa.

---

## Resumo comparativo: RC9 vs master

| Comportamento | v7.0.0-rc.9 | master |
|--------------|-------------|--------|
| `processingMutex` | 1 mutex global (serialização total) | 4 mutexes independentes (concorrência entre tipos) |
| `makeMutex` implementação | Promise-chain manual (erros engolidos, fila nunca trava) | `async-mutex` (comportamento correto mas diferente) |
| `presenceSubscribe` assinatura | `(jid, tcToken?)` síncrona | `async (jid)` — lê tcToken da store |
| `end()` | síncrona | `async` — aguarda `ws.close()` |
| `mapWebSocketError` tipo | `(err: Error) => void` ✓ | igual, mas recebe função `async` sem `await` ⚠️ |
| `sendUnifiedSession` | não existia | chamada após login E após cada presence `available` |
| `saveIdentity` antes de pkmsg | não existia | sim — transação por mensagem pkmsg recebida |
| `shouldRecreateSession` desde retry 1 | sim | não — só a partir do retry 2 |
| Verificação cert TLS leaf | ignorada (TODO) | verificação criptográfica com chave hardcoded |
| `hkdf` / `md5` | WebCrypto assíncrono (node built-in) | Rust WASM síncrono (`whatsapp-rust-bridge`) |
| `shouldSyncHistoryMessage` padrão | `() => !!syncFullHistory` | `syncType !== FULL` (ignora `syncFullHistory`) |
| `handleMessage` processado sob mutex | sim — aguarda recibos e notificações | não — apenas messageMutex, outros correm em paralelo |
| `profilePictureUrl` / `presenceSubscribe` | envia diretamente | lê tcToken da store antes de enviar |

---

## Como diagnosticar qual causa específica está afetando você

### Passo 1: Veja o log imediatamente antes de cada 428

```
[baileys] connection: open
[baileys] failed to send unified_session telemetry  ← ou sem este log
[baileys] Connection Terminated  ← ws.on('close')
[baileys] connection: close  statusCode: 428
```

Se o 428 vem logo após `connection: open` (menos de 1-2 segundos), o candidato mais
forte é `sendUnifiedSession`.

### Passo 2: Veja se é sempre na PRIMEIRA ação após conectar

Se a primeira chamada a `sendMessage` ou `groupMetadata` sempre retorna 428, o
WebSocket fechou durante o período de inicialização. Verifique:
- `sendUnifiedSession` (ocorre imediatamente após login)
- Verificação de certificado TLS (`noise certificate signature invalid`)
- Inicialização do WASM (`whatsapp-rust-bridge` instalado?)

### Passo 3: Veja se o 428 acontece sob carga (muitas mensagens simultâneas)

Se o 428 ocorre quando há muitas mensagens chegando ao mesmo tempo (ex.: grupos
movimentados), pode ser a mudança dos 4 mutexes concorrentes causando estado
inconsistente.

### Passo 4: Verifique se usa `syncFullHistory: false`

Se sim, defina explicitamente `shouldSyncHistoryMessage: () => false` para restaurar
o comportamento da RC9:
```ts
makeWASocket({
    syncFullHistory: false,
    shouldSyncHistoryMessage: () => false,
})
```

### Passo 5: Verifique a versão do Node.js e instalação do WASM

```bash
node -e "require('whatsapp-rust-bridge'); console.log('WASM OK')"
```

Se falhar, reinstale as dependências:
```bash
npm install
# ou
yarn install
```

---

## Workarounds temporários por causa

| Causa | Workaround |
|-------|-----------|
| `sendUnifiedSession` causa 428 | Verificar se o servidor rejeita o nó `ib/unified_session` — pode ser necessário remover as chamadas `void sendUnifiedSession()` em `socket.ts` e `chats.ts` |
| `syncFullHistory: false` não funciona mais | Adicionar `shouldSyncHistoryMessage: () => false` explicitamente |
| WASM não inicializa | Reinstalar deps: `npm install` / verificar Node.js ≥ 16 com suporte a WASM |
| 428 sob carga de mensagens | Aguardar fix oficial do mutex ou contribuir com PR revertendo para mutex único |
| Certificado TLS inválido | Verificar se a chave pública hardcoded `142375574d0a587166aae71ebe516437c4a28b73e3695c6ce1f7f9545da8ee6b` ainda é válida para o ambiente |

---

*Gerado em: 2026-03-11 | Comparação: v7.0.0-rc.9 → master (HEAD: 76aa555)*
