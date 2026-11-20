# M3 — Mini-sistema de arquivos em memória (em C)

Simulador de um sistema de arquivos Unix-like que roda **inteiro em memória**, sem tocar no sistema de arquivos real:

- árvore de diretórios com `struct` e ponteiros;
- um **FCB/inode** por arquivo;
- permissões **RWX (dono/grupo/outros)** checadas com bitmask em toda operação;
- **disco simulado** (`char[]`) em blocos, com **alocação indexada**.

Há dois modos de uso, que rodam o mesmo código: um **menu numerado** (inspirado em [eldanielG/mini_fs_linux](https://github.com/eldanielG/mini_fs_linux)) e um **terminal** com comandos do Linux (`ls -l`, `chmod 640`, `echo "x" > arq`...).

| Arquivo | Conteúdo |
| --- | --- |
| `src/main.c` | menu e escolha do modo |
| `src/shell.c/.h` | terminal: interpreta comandos e imprime resultados/erros |
| `src/fs.c/.h` | árvore de diretórios, caminhos e operações |
| `src/inode.c/.h` | FCB/inode, tabela de inodes e alocação indexada |
| `src/perm.c/.h` | usuários, grupos, RWX e `chmod` |
| `src/disco.c/.h` | disco simulado e bitmap de blocos livres |
| `src/erros.c/.h` | códigos e mensagens de erro |
| `demo.txt` | roteiro da apresentação |

## 1. Compilar e executar

Precisa de `gcc` e `make` (e `valgrind` para o `memcheck`). Funciona em Linux, WSL ou Codespaces.

```bash
make            # compila
./mfs           # menu numerado
./mfs -t        # terminal
make demo       # roda o roteiro demo.txt
make memcheck   # roda o roteiro no valgrind (0 erros, 0 leaks)
```

O terminal também aceita um roteiro pela entrada padrão: `./mfs -t < roteiro.txt`.

No **menu**, cada opção pergunta os dados e chama o mesmo interpretador do terminal, então erros e permissões são idênticos nos dois modos. A opção 24 abre o terminal; `menu` volta.

### Estado inicial

| Usuário | uid | Grupo (gid) |
| --- | --- | --- |
| `root` | 0 | root (0) |
| `alice` | 1000 | dev (100) — **usuário inicial**, em `/home/alice` |
| `bob` | 1001 | dev (100) |
| `carol` | 1002 | visitantes (200) |

Diretórios: `/home/{alice,bob,carol}` (755, cada um do seu dono) e `/tmp` (777). `su` troca de usuário sem senha.

### Comandos

| Comando | O que faz |
| --- | --- |
| `pwd`, `cd [dir]` | diretório atual / navegar |
| `ls [-l] [-i] [caminho...]`, `tree [dir]` | listar / desenhar a árvore |
| `mkdir [-p]`, `rmdir`, `touch` | criar/remover diretório, criar arquivo |
| `echo [-e] [-n] texto > arq` / `>> arq` | escrever / acrescentar |
| `cat arq`, `read arq offset n` | ler tudo / ler `n` bytes a partir de `offset` |
| `truncate -s tamanho arq` | truncar ou estender com zeros |
| `cp`, `mv`, `ln` | copiar / mover / hard link |
| `rm [-r] [-f]` | remover |
| `stat caminho` | mostrar o FCB completo |
| `chmod modo arq`, `chown usuario[:grupo] arq`, `umask` | permissões (octal ou `u+x`, `go-w`...) |
| `exec prog` ou `./prog` | executar um script (`#!`) com comandos do simulador |
| `su`, `whoami`, `id` | usuário atual |
| `blocos arq`, `df`, `inodes` | blocos do arquivo, uso do disco, tabela de inodes |
| `help`, `exit` | |

## 2. Arquitetura

```
main.c (menu) ──► shell.c (terminal) ──► fs.c (árvore e operações)
                                           ├─► perm.c  (RWX)
                                           └─► inode.c (FCB e blocos) ──► disco.c (blocos + bitmap)
```

- As dependências só descem: `disco` não conhece arquivos, `inode` não conhece diretórios, `fs` não conhece o terminal.
- Só o shell imprime; as outras camadas devolvem um `FsErro`.
- Todo acesso passa por uma única função: `perm_checa`.
- Sem variáveis globais: todo o estado fica em uma `struct Fs` passada por ponteiro.

## 3. Conceitos e implementação

### Arquivo, atributos e FCB

```c
typedef struct {
    int         id;                 /* número do inode */
    TipoArquivo tipo;               /* diretório, texto, numérico, binário, programa */
    size_t      tamanho;
    int         dono, grupo;
    uint16_t    perm;               /* rwxrwxrwx, ex.: 0754 */
    time_t      criado, modificado, acessado;
    int         n_links;
    int         n_blocos;
    int         diretos[8];         /* ponteiros diretos */
    int         indireto;           /* bloco com mais 16 ponteiros */
} FCB;
```

- O **nome não fica no FCB**, e sim na entrada do diretório (como no Unix). Por isso `mv` mantém o inode, `ln` cria outro nome para o mesmo inode e `rm` só libera os dados quando `n_links` chega a 0.
- O **tipo** é deduzido do conteúdo a cada escrita (`#!` → programa, bytes de controle → binário, só dígitos → numérico, senão texto).
- A **tabela de inodes** é um vetor dinâmico; números liberados são reaproveitados.

### Operações

| Operação | Comando | Função |
| --- | --- | --- |
| criar | `touch` | `fs_touch` |
| escrever | `echo >`, `>>` | `fs_escreve` → `inode_escreve` |
| ler | `cat` | `fs_le` → `inode_le` |
| reposicionar | `read arq offset n` | `fs_le(offset, n)` |
| excluir | `rm` | `fs_rm` |
| truncar | `truncate -s` | `fs_trunca` → `inode_trunca` |

`cp` cria novo inode e novos blocos; `mv` só move a entrada; `ln` cria nova entrada para o mesmo inode (`n_links++`). Dá para ver isso com `ls -i`.

### Diretórios em árvore

```c
typedef struct Entrada {
    char            nome[NOME_MAX + 1];
    FCB            *inode;
    struct Entrada *pai, *filhos, *irmao;
} Entrada;
```

Árvore n-ária (primeiro filho / próximo irmão), com filhos em ordem alfabética. O ponteiro `pai` permite `cd ..`, montar o `pwd` e impedir `mv dir dir/sub` e `rmdir` do diretório atual. Caminhos podem ser absolutos ou relativos; `.` e `..` são resolvidos na hora.

Vantagens sobre diretório de nível único ou de dois níveis: busca mais rápida, mesmo nome em diretórios diferentes e agrupamento de arquivos relacionados. Com `ln` a estrutura vira um **grafo acíclico**; hard link para diretório é proibido para evitar ciclos.

### Proteção (RWX com bitmask)

```
         dono   grupo  outros
         r w x  r w x  r w x       chmod 754 = 111 101 100 = rwxr-xr--
```

```c
int desloc = (u->uid == f->dono)  ? 6
           : (u->gid == f->grupo) ? 3 : 0;
return ((f->perm >> desloc) & op) == op;
```

`chmod` aceita octal e simbólico (`u+x` → `|= 0100`, `go-w` → `&= ~0022`). O `umask` (padrão 022) é aplicado na criação.

| Operação | Exige |
| --- | --- |
| atravessar diretório, `cd` | `x` no diretório |
| `ls` | `r` no diretório |
| `cat`, `read`, `cp` (origem) | `r` no arquivo |
| `echo`, `truncate` | `w` no arquivo |
| criar/apagar nome (`touch`, `mkdir`, `rm`, `mv`, `ln`) | `w` e `x` no diretório pai |
| `exec` | `r` e `x` no arquivo |
| `chmod` / `chown` | ser dono ou root / ser root |

Como no Linux: o dono usa só os bits de dono (com `chmod 077` a própria dona é negada); diretório `711` não pode ser listado, mas seus arquivos podem ser abertos pelo nome; root lê e escreve tudo, mas só executa se houver algum bit `x`.

### Alocação de blocos

Disco de 256 blocos × 64 bytes, com **bitmap** de livres e alocação *first-fit*.

| Método | Vantagem | Desvantagem |
| --- | --- | --- |
| Contígua | acesso direto simples | fragmentação externa |
| Encadeada | sem fragmentação externa | acesso sequencial lento |
| **Indexada** (usada) | acesso direto, sem fragmentação externa | espaço com índices, tamanho máximo |

Estilo ext2: **8 ponteiros diretos + 1 bloco indireto** (16 ponteiros). Tamanho máximo: (8 + 16) × 64 = **1536 bytes**.

```
byte p → bloco lógico i = p / 64, posição p % 64
i < 8  → diretos[i]
i >= 8 → (i - 8)-ésimo ponteiro do bloco indireto
```

Antes de alterar, o simulador confere se há espaço; se não houver, o arquivo fica intacto. Os comandos `blocos`, `stat` e `df` mostram o mapeamento e o mapa de bits.

## 4. Erros e memória

Os erros são um `enum FsErro` e saem no formato do Linux, `comando: alvo: mensagem` (ex.: `cat: nada: arquivo ou diretório não encontrado`, `chmod: nota.txt: operação não permitida`). Entradas inválidas, aspas abertas e scripts recursivos (limite de 8 níveis) são tratados sem derrubar o programa.

- Cada alocação tem um único dono responsável por liberá-la; `fs_destroi` libera tudo no final.
- Todo `malloc`/`calloc`/`realloc` é verificado e desfeito em caso de falha.
- Entrada limitada: `fgets`, nomes até 32 caracteres, caminhos até 255.
- Compila com `-std=c11 -Wall -Wextra -Wpedantic` sem warnings; `make memcheck` dá 0 erros e 0 leaks.

## 5. Comparação com o Linux

Os mesmos cenários (`ln`, `mv`, `cp` com `ls -i`; leitura/escrita por dono, grupo e outros; `chmod 077`; diretório 711; `chmod u+x,go-w`) dão os mesmos resultados no Ubuntu e no simulador, mudando só o idioma das mensagens.

Simplificações: diretórios não ocupam blocos (tamanho 0 no `ls -l`); não há links simbólicos, `cp -r`, setuid/sticky bit nem senha no `su`; `exec` interpreta comandos do simulador em vez de executar código real.

## 6. Créditos

- Menu numerado inspirado em [eldanielG/mini_fs_linux](https://github.com/eldanielG/mini_fs_linux) (que usa o sistema de arquivos real; aqui tudo é simulado).
- Enunciado: disciplina de Sistemas Operacionais, Univali.
