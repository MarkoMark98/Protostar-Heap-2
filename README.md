# Protostar – Heap2

Progetto di analisi ed exploit della challenge **Heap2** della VM didattica **Protostar** (Exploit Education), realizzato da **Marco Marchionno** e **Leonardo Monaco**.

## Obiettivo

La sfida Heap2 è completa quando il programma stampa:

```text
You have logged in alredy!
```

Il binario gestisce una struct `auth` sull’heap e il puntatore globale `auth`.  
Non esistono funzioni “legittime” per impostare `auth->auth`: occorre quindi **corrompere l’heap** per far risultare il campo `auth` non nullo al momento del comando `login`.

## Ambiente

- VM: Protostar (`exploit-exercises-protostar-2.iso`)
- OS: Debian GNU/Linux 6.0.3 (squeeze), x86 32 bit, little endian
- Utenti:
  - attaccante: `user:user`
  - root: `root:godmode`
- Binario: `/opt/protostar/bin/heap2`

## Vulnerabilità principali

Nel codice semplificato:

```c
struct auth {
    char name; [fmathis](http://www.fmathis.com/publications/mathis2020_tochi.pdf)
    int auth;
};

struct auth *auth;
char *service;
```

1. **Allocazione insufficiente (CWE‑131)**

```c
auth = malloc(sizeof(auth));      // alloca solo 4 byte
memset(auth, 0, sizeof(auth));
```

- Vengono allocati 4 byte (size del puntatore), ma il codice accede a `name[32]` e `auth`.
- Causa overflow su heap nella struct.

2. **Use After Free (CWE‑416)**

```c
if (!strncmp(line, "reset", 5))
    free(auth);                   // auth non viene azzerato
```

- La memoria viene liberata ma il puntatore resta valido.
- Iterazioni successive possono sfruttare la stessa area di heap.

3. **`service = strdup(line + 7)` come primitiva di sovrascrittura**

- `strdup` alloca nuovo chunk e copia l’input.
- Se usato dopo `auth`/`reset`, permette di **riempire l’area di memoria dove risiede (o risiedeva) la struct**.

## Tecniche di exploit

### 1. Use After Free

Sequenza logica:

1. `auth AAA`
2. `reset`
3. `service <payload>`
4. `login`

Idea:

- `auth` punta a un chunk liberato.
- `service`/`strdup` riusa quel chunk e copia una stringa abbastanza lunga da raggiungere la posizione di `auth->auth` (es. 32 byte di padding).
- `auth->auth` diventa non nullo → `login` stampa il messaggio di successo.

### 2. Allocazione insufficiente

Sequenza logica:

1. `auth AAA`
2. `service <payload>`
3. `login`

Idea:

- `auth` alloca solo 4 byte.
- Il chunk di `service` viene posizionato immediatamente dopo e, con un payload di lunghezza opportuna, va a **sovrascrivere il campo `auth` della struct**.
- Anche qui, `auth->auth` diventa non nullo e il `login` va a buon fine.

## Mitigazioni proposte

- **Use After Free**:
  ```c
  if (!strncmp(line, "reset", 5)) {
      free(auth);
      auth = NULL;
  }
  ```

- **Allocazione insufficiente**:
  ```c
  auth = malloc(sizeof(struct auth));
  memset(auth, 0, sizeof(struct auth));
  ```

## Struttura consigliata del repo

```text
heap2-protostar/
├── Heap2-Marchionno-Monaco.pdf   # slide/relazione
└── README.md                     # questo file
```
