# Trabalho – Conceitos de Sistema Operacional (Processos, IPC, Memória e Arquivos)

Autor: Gabriel W. Petermann  
Data: 14/11/2025 

---

## 1. Monitoramento de Processos

O script `monitor_processos.sh` lista os 5 processos que mais consomem CPU e memória, exibindo: PID, usuário, %CPU, %MEM, tempo de execução e comando. Também gera alertas se algum processo ultrapassa limites pré-definidos de uso de CPU ou memória.

### Script
```bash
#!/usr/bin/env bash
LIMITE_CPU=50.0
LIMITE_MEM=30.0
echo "Top 5 processos por uso de CPU:"
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head -n 6
echo ""
echo "Top 5 processos por uso de MEMÓRIA:"
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%mem | head -n 6
echo ""
echo "Verificando alertas..."
ps -eo pid,%cpu,%mem,cmd --sort=-%cpu | awk -v c="$LIMITE_CPU" -v m="$LIMITE_MEM" '
NR>1 {
  alerta=""
  if ($2 > c) alerta=alerta "[ALTO USO CPU] "
  if ($3 > m) alerta=alerta "[ALTO USO MEM] "
  if (alerta!="") {
    printf "%sPID=%s CPU=%.1f%% MEM=%.1f%% CMD=%s\n", alerta, $1, $2, $3, $4
  }
}'
```

### Execução
```bash
chmod +x scripts/monitor_processos.sh
./scripts/monitor_processos.sh
```

### Captura

<img width="801" height="725" alt="image" src="https://github.com/user-attachments/assets/2a8a931b-b421-4576-a461-9afe9eac8fa0" />

---

## 2. Comunicação entre Processos (IPC via Arquivos)

Dois scripts simulam a comunicação:
- `escritor.sh`: gera mensagens a cada 5 segundos em `/tmp/mensagens.txt`
- `leitor.sh`: lê mudanças e registra respostas em `/tmp/respostas.txt`

### Scripts
```bash
# escritor.sh
#!/usr/bin/env bash
ARQ="/tmp/mensagens.txt"
while true; do
  echo "$(date '+%H:%M:%S') - Mensagem gerada pelo escritor" >> "$ARQ"
  sleep 5
done
```

```bash
# leitor.sh
#!/bin/bash
set -Eeuo pipefail

ARQ_MSG="/tmp/mensagens.txt"
ARQ_RESP="/tmp/respostas.txt"

touch "$ARQ_MSG" "$ARQ_RESP"
ULTIMO_TAM=0

echo "Iniciando leitor..."
while true; do
  if [ -f "$ARQ_MSG" ]; then
    TAM_ATUAL=$(stat -c%s "$ARQ_MSG")
    if [ "$TAM_ATUAL" -ne "$ULTIMO_TAM" ]; then
      NOVAS=$(tail -n 5 "$ARQ_MSG")
      printf '%s - Lidas: %s\n' "$(date '+%H:%M:%S')" "$NOVAS" >> "$ARQ_RESP"
      ULTIMO_TAM=$TAM_ATUAL
    fi
  fi
  sleep 3
done
```

### Execução
```bash
chmod +x scripts/escritor.sh scripts/leitor.sh
./scripts/escritor.sh    # Terminal 1
./scripts/leitor.sh      # Terminal 2
tail -f /tmp/mensagens.txt
tail -f /tmp/respostas.txt
```

### Capturas

#### Arquivo Escritor rodando

<img width="586" height="36" alt="image" src="https://github.com/user-attachments/assets/cfbf24ec-c2d0-4798-9889-c86959b0538c" />

#### Arquivo Leitor rodando

<img width="578" height="46" alt="image" src="https://github.com/user-attachments/assets/0c7bc706-5a0a-413c-918a-206f6831c74a" />

#### Arquivos de Mensagem

<img width="921" height="311" alt="image" src="https://github.com/user-attachments/assets/b90e5afc-54c5-4d0f-9fa1-58ec1fd67797" />


---

## 3. Exercício de Memória (Alocação Gradual)

O programa em C `aloca_memoria.c` aloca blocos de 400 MB e imprime o total acumulado até falhar (falta de memória), permitindo observar crescimento de uso de RAM e possível ativação de swap.  
O script `monitor_memoria.sh` observa em loop o uso de CPU/MEM do processo e a situação geral da memória (RAM / Swap).

### Código C
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define BLOCO_MB 400
#define SEGUNDOS_PAUSA 3

int main() {
    size_t bloco = BLOCO_MB * 1024 * 1024;
    size_t total = 0;
    void **vetor = NULL;
    int capacidade = 0;

    while (1) {
        void *mem = malloc(bloco);
        if (!mem) {
            fprintf(stderr, "Falha de alocação após %.2f GB\n", total / (1024.0*1024*1024));
            break;
        }
        capacidade++;
        vetor = realloc(vetor, capacidade * sizeof(void*));
        vetor[capacidade - 1] = mem;
        total += bloco;
        printf("Alocado total: %.2f GB\n", total / (1024.0*1024*1024));
        fflush(stdout);
        sleep(SEGUNDOS_PAUSA);
    }
    printf("Pressione Ctrl+C para encerrar.\n");
    while (1) sleep(10);
}
```

### Compilação e Execução
```bash
gcc src/aloca_memoria.c -o bin/aloca_memoria
./bin/aloca_memoria
pidof aloca_memoria
```

### Monitoramento
```bash
./scripts/monitor_memoria.sh <PID>
```

### Script de Monitoramento
```bash
#!/usr/bin/env bash
PID="$1"
if [ -z "$PID" ]; then
  echo "Uso: $0 <PID_DO_PROCESSO>"
  exit 1
fi
echo "Monitorando PID $PID (Ctrl+C para sair)"
while true; do
  echo "===== $(date '+%H:%M:%S') ====="
  ps -p "$PID" -o pid,user,%cpu,%mem,etime,rss,cmd
  free -h
  grep -E 'Mem|Swap' /proc/meminfo | head -n 4
  sleep 5
done
```

### Capturas

#### Alocação de memória

<img width="447" height="395" alt="image" src="https://github.com/user-attachments/assets/e602684c-3468-488f-a4b5-9b428a9ff0b9" />

#### Uso de swap

<img width="577" height="572" alt="image" src="https://github.com/user-attachments/assets/de450768-1498-4f84-b6bf-57ef878a6dc1" />


---

## 4. Gerenciamento de Arquivos e Permissões

O script `gerencia_arquivos.sh` cria a árvore:
```
~/sistema_operacional/
  docs/
  src/
  bin/
  logs/
```
Aplica permissões diferentes e lista recursivamente, depois mostra arquivos modificados nas últimas 24h.

### Script
```bash
#!/usr/bin/env bash
BASE=~/sistema_operacional
DIRS="docs src bin logs"
echo "Criando estrutura em $BASE"
mkdir -p "$BASE"
for d in $DIRS; do
  mkdir -p "$BASE/$d"
done
echo "Criando arquivos de exemplo..."
echo "Documento de teste" > "$BASE/docs/README.txt"
echo "#include <stdio.h>" > "$BASE/src/exemplo.c"
echo "log inicial" > "$BASE/logs/app.log"
touch "$BASE/bin/script.sh"
chmod 644 "$BASE/docs/README.txt"
chmod 600 "$BASE/logs/app.log"
chmod 755 "$BASE/bin/script.sh"
chmod 644 "$BASE/src/exemplo.c"
echo "Estrutura criada."
echo "Listagem detalhada:"
ls -lR "$BASE"
echo "Arquivos alterados últimas 24h:"
find "$BASE" -type f -mtime -1 -printf '%TY-%Tm-%Td %TH:%TM:%TS %p\n'
```

### Execução
```bash
chmod +x scripts/gerencia_arquivos.sh
./scripts/gerencia_arquivos.sh
```

### Capturas

#### Estrutura e permissões

<img width="559" height="500" alt="image" src="https://github.com/user-attachments/assets/476f01ae-8a18-450e-8717-ac210c59dd23" />


---

## Conclusões

Este trabalho me ajudou a compreender:
1. Monitoramento de processos com filtros e alertas.
2. Comunicação simples entre processos via arquivos (IPC).
3. Impacto da alocação de memória e ativação de swap.
4. Criação e administração de diretórios, arquivos e permissões.

Cada etapa reforça conceitos fundamentais de sistemas operacionais: gerenciamento de recursos, sincronização básica, uso de memória e organização do sistema de arquivos.
