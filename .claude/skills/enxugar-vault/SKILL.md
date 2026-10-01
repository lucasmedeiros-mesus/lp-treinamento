---
name: enxugar-vault
description: Audita um vault Obsidian (ou qualquer pasta de notas usada com Claude Code) e corta o custo de tokens SEM perder contexto. Mede o que é lido toda sessão, acha a gordura, propõe o plano, aplica com backup e instala as regras que impedem a gordura de voltar. Use quando o usuário disser "meu vault tá caro/pesado", "gasto muito token", "como diminuir o custo do Claude no meu segundo cérebro", "enxuga meu vault", "/enxugar-vault". Funciona em vault de QUALQUER pessoa — não assume estrutura nenhuma.
---

# /enxugar-vault — Dieta de tokens do segundo cérebro

Corta o custo de contexto de um vault **sem perder informação**. O ganho típico é **30–40% na abertura
de sessão** (custo que se paga em TODA conversa, pra sempre) + toneladas de texto morto fora do caminho.

⛔ **Esta skill NÃO apaga nada.** Ela move, comprime e aponta. Todo bruto vai pra `~/backups/`.
⛔ **NUNCA aplicar sem o OK explícito do usuário.** Mede → propõe → espera → aplica.

---

## As 5 leis (a teoria que faz o resto funcionar)

1. **Token é TEXTO lido, não arquivo em disco.** Imagem, vídeo e PDF não entram no contexto. Uma pasta de
   1 GB de mídia pode custar 0 token. **Nunca meça com `du -sh`** — meça o peso dos `.md` que são LIDOS.
2. **Custo recorrente vale 10x o custo frio.** Arquivo lido em TODA sessão (`CLAUDE.md`, `MEMORY.md`, o
   "painel do agora" da pessoa) é onde 1 KB economizado se multiplica por cada conversa do resto da vida.
   Arquivo grande que ninguém lê custa **zero** — não perca tempo com ele (mas diga isso ao usuário, senão
   ele acha que arquivar aquilo economizou algo).
3. **Tag sozinha não economiza nada.** Encher o arquivo de `<contexto>...</contexto>` e continuar lendo ele
   inteiro = **zero**. O `Read` carrega o arquivo todo. O que economiza é **âncora + regra de leitura**:
   `grep` acha a linha da seção → `Read` com offset/limit lê só aquele range. **E a maioria dos vaults já
   tem as âncoras: são os headers `##`.** Só falta a regra.
4. **Comprimir ≠ deletar.** Transcrição/log velho vira **resumo + link pro bruto** no `~/backups/`.
   Informação preservada, token liberado.
5. **Gordura costuma esconder bug.** Doc inchado acumula instrução contraditória (ex.: manda usar a cor que
   ele mesmo proíbe 100 linhas abaixo). Sanear economiza token **e** conserta o comportamento do Claude.

---

## FASE 1 — MEDIR (nunca chutar)

Rodar no diretório do vault. Adaptar os nomes de pasta ao que existir de fato.

```bash
# 1. Peso de TEXTO por pasta (o que vira token) — ignora mídia de propósito
for d in */; do
  sz=$(find "$d" -name '*.md' -type f -exec cat {} + 2>/dev/null | wc -c)
  n=$(find "$d" -name '*.md' -type f 2>/dev/null | wc -l)
  printf "%8d KB  %4d arq  %s\n" $((sz/1024)) "$n" "$d"
done | sort -rn

# 2. Os monstros (candidatos a comprimir/splitar/leitura cirúrgica)
find . -name '*.md' -type f -not -path './.obsidian/*' -size +15k -exec ls -la {} + | sort -k5 -rn | head -20

# 3. O HOT-PATH — o ouro. Separar o que é SEMPRE lido do que é condicional.
find . -name 'CLAUDE.md' -o -name 'MEMORY.md' | xargs wc -c | sort -rn
```

**Distinguir os dois níveis (não confundir — muda o alvo):**

| Nível | O que é | Vale |
|---|---|---|
| 🔴 **Sempre lido** | `CLAUDE.md` da **raiz** + `~/.claude/CLAUDE.md` global + o `MEMORY.md` de auto-memória + **o painel do agora** | **10x** — entra em toda conversa |
| 🟡 **Condicional** | `CLAUDE.md`/`MEMORY.md` **de pasta** (só carregam ao mexer naquela pasta) | 1x-ish — enxugar ajuda, mas não é o ouro |

**Achar o "painel do agora" da pessoa:** ler o `CLAUDE.md` raiz e o global procurando regra do tipo
*"leia X primeiro"* / *"sempre consulte Y"*. **É quase sempre o maior ofensor** e o nome varia
(`agora.md`, `hoje.md`, `dashboard.md`, `status.md`, `README.md`…). Se não houver regra dessas, o hot-path
é só CLAUDE.md + memória.

⚠️ **Somar só o nível 🔴 como "custo de abrir sessão".** Somar os condicionais infla o número e a promessa
vira mentira. Reporte os dois separados.

## FASE 2 — DIAGNOSTICAR

Classificar cada ofensor em **RECORRENTE** (hot-path) ou **FRIO** (só lido sob demanda), e dar uma ação:

| Ação | Quando | Como |
|---|---|---|
| **PODAR** | painel do agora / índice com item já resolvido | item fechado → arquivo de histórico; deixa só o vivo |
| **COMPRIMIR** | transcrição, log, dump longo já superado | resumo+link no vault, **bruto pro `~/backups/`** |
| **ARQUIVAR** | mês/projeto encerrado | move pra `arquivo/` (sai do caminho, continua buscável) |
| **SPLITAR** | doc grande que mistura assuntos | 1 arquivo por assunto + índice que aponta |
| **LEITURA CIRÚRGICA** | doc grande **vivo** que precisa ficar inteiro | não mexe no conteúdo — instala a regra do grep (Fase 5) |
| **DEDUPLICAR** | mesmo fato em 3 lugares / corpo repetindo frontmatter | fonte de verdade única, resto aponta |
| **MANTER** | grande mas fora do caminho de leitura | **dizer que é 0 token** — não gastar trabalho |

Procurar também: **contradição** (duas instruções brigando), **marca/nome legado** e **arquivo vazio**.

## FASE 3 — PROPOR (e PARAR)

Entregar UMA tabela, ordenada por retorno: item · KB · recorrente ou frio · ação · economia estimada.
Separar **🔴 recorrente** de **🟡 frio** — e explicar que o recorrente vale 10x.
Fechar com **1 quick-win** (o de maior retorno e menor risco) e **perguntar o que aplicar**.

⛔ Não aplicar nada aqui. Nem "só o quick-win porque é óbvio".

## FASE 4 — APLICAR (só o que foi aprovado)

**Backup ANTES de qualquer corte, sempre:**
```bash
mkdir -p ~/backups/enxugar-vault-$(date +%F)
cp -R <pasta-alvo> ~/backups/enxugar-vault-$(date +%F)/
```

Regras de execução:
- **Comprimir transcrição:** cortar da seção de transcrição pra baixo (`awk '/^## Transcri/{exit} {print}'`),
  deixando cabeçalho + resumo + decisões + ações + link da gravação; carimbar onde foi parar o bruto.
- **Nunca comprimir o que está vivo** — cliente/projeto ativo, mentorado em curso, mês corrente. **Pergunte
  qual é o vivo, não deduza.** Na dúvida, mantenha.
- **Depois de mover, checar link quebrado:** `grep -rn "pasta-antiga/arquivo" --include='*.md' .`
  (No Obsidian, link por nome `[[nota]]` sobrevive à mudança de pasta; link com caminho `[[pasta/nota]]` quebra.)
- Ir **item por item**, mostrando o antes/depois em KB.

## FASE 5 — BLINDAR (o passo que ninguém faz — e é o que dura)

Sem isso a gordura volta em 3 semanas. Instalar no `CLAUDE.md` do vault:

**1. Regra de leitura cirúrgica** (a versão que funciona da ideia de bloco/tag):
```markdown
## Leitura cirúrgica (economia de contexto)

Arquivo `.md` com mais de ~15 KB (fora os lidos inteiros por regra: `CLAUDE.md`, `MEMORY.md` e o painel
do agora): **nunca ler inteiro de cara.** Primeiro mapear com `grep -n "^#" arquivo.md` (os headers são as
âncoras), depois `Read` **só o range** da seção necessária (offset/limit). Se a seção não tiver header,
achar por palavra-chave com `grep -n`. Ler inteiro só quando a tarefa exigir o documento todo.
```
> Se o doc grande **não tiver headers**, aí sim vale inserir âncoras — pode ser `## Header` normal (melhor,
> porque o Obsidian também navega) ou `<bloco id="x">` se a pessoa preferir. A tag é opcional; **a regra não é.**

**2. Estrutura do painel do agora** — 3 seções fixas: `vivo agora` → `decisões/negociações em aberto` →
`pendências, 1 linha cada`. Regra: **bloco fechado sai na hora** pro histórico.

**3. Rotação de log** (se houver pasta de diário/log que só cresce):
```markdown
- Mês fechado há >60 dias → mover pra `arquivo/{pasta}/YYYY-MM/`.
```

**4. Registrar o estado** no `MEMORY.md` das pastas tocadas (data no topo) — curadoria, não empilhamento.

---

## Fechamento

Entregar o placar: **abertura de sessão antes → depois** (é a métrica que importa), total de texto tirado
do caminho, onde estão os backups, e o que ficou pro usuário decidir. Lembrar do `/tldr` se o vault tiver
o rito de log de sessão.

## Erros que já custaram caro (não repetir)

- ❌ Medir com `du -sh` e sair "otimizando" pasta de imagem → **0 token economizado**.
- ❌ Taggear arquivo e continuar lendo inteiro → **0 token economizado**.
- ❌ Comprimir transcrição de cliente/mentorado **ativo** → perda de contexto real.
- ❌ Aplicar sem backup ou sem OK → destrutivo e irreversível.
- ❌ Otimizar arquivo frio e vender como economia → é 0; seja honesto no relatório.
