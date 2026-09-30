# CLAUDE.md · cnataquara-escape

Instruções permanentes para sessões do Claude neste repo.

## O que é

Escape room virtual do CNA, "O Sumiço da Dani" (nasceu no CNA Taquara). Peça de marketing e
captação de leads, com 5 salas que seguem o layout físico da escola:
Recepção, Garden, Sala de Espera, Sala de Aula, Auditório.

- Produção: https://escape.cnataquara.com.br
- Supabase: `gpnwmsnayrqjcmhqrtpx` (o **mesmo** do CRM)
- Nome do repo segue o padrão da unidade: `cnataquara-{subdomínio}`

## Arquitetura

Monólito de arquivo único. `index.html` com ~774 linhas, **um único bloco
`<script>`**, sem build.

```
index.html          jogo completo
assets/dani.png     mascote (405 KB, maior arquivo do repo)
assets/logo-cna.png
CNAME               escape.cnataquara.com.br
```

Acesso a dados via REST direto, sem `supabase-js`:

```js
const SB_URL='https://gpnwmsnayrqjcmhqrtpx.supabase.co';
function rpc(...)   // helper único, só RPC
```

Escrita e timing passam por RPC. Leitura direta só de `crm_unidades`,
`escape_temporadas`, `escape_salas` e das views de ranking.

### Base única multiunidade (desde 30/09/2026)

O banco `gpnwmsnayrqjcmhqrtpx` ("CRM CNA") é a base única da rede. Tabela de
dado de unidade tem coluna `unidade` (slug de `crm_unidades`), inclusive as
`escape_*`. Globais, sem `unidade`: `crm_unidades`, `crm_modulos`,
`crm_papel_permissoes`, `crm_usuarios`, `crm_sessoes`.

- A página descobre a unidade por `?unidade=<slug>`, validada em
  `crm_unidades` (status diferente de `inativa`). Sem parâmetro ou com slug
  inválido: `taquara`, para os links e QR codes antigos continuarem valendo.
- `escape_iniciar` recebe `p_unidade`; `escape_salas` é filtrada por unidade.
- As views de ranking não têm `unidade`: o ranking por sala filtra por
  `sala_id` (das salas da unidade) e o geral por `temporada_id` (temporadas da
  unidade).
- Nome exibido vem de `crm_unidades.nome`. **Proibido `if` por nome ou slug de
  unidade.** Comportamento por unidade vem do banco.

No CRM, login é por vínculo (`crm_usuario_unidades`) e `crm_definir_senha` está
depreciada (use `crm_equipe_definir_senha`).

## Timing é server-side, e isso não é negociável

Ranking com timing calculado no cliente é manipulável. As RPCs calculam tempo
decorrido a partir de timestamp do banco, com penalidade aplicada no servidor:

`escape_iniciar`, `escape_sala_iniciar`, `escape_sala_concluir`,
`escape_sala_estourar`, `escape_sessao_concluir`.

Leitura: `escape_salas`, `escape_ranking_sala`, `escape_ranking_geral`.

Nunca mova cálculo de tempo ou de penalidade para o cliente, nem "só para
testar".

## Atribuição de lead

O parâmetro `qr` da URL entra por `URLSearchParams`. Cada cartaz da campanha de
guerrilha ("desaparecida", com QR code) usa um `origem` próprio, para
atribuição no CRM.

Separação que importa:

- Jogador **não aluno** vira lead em `crm_leads` com `origem = 'Escape Room'`
- Jogador **aluno** vai para `escape_jogadores`, fora do funil comercial

Ranking é unificado entre os dois. Funil não é.

## Convivência com o CRM

Mesmo banco, prefixos diferentes: `escape_*` aqui, `crm_*` no CRM. Não misture.
O projeto Supabase é nomeado por domínio de dado (comercial), não pelo primeiro
app que chegou nele.

RLS: policy permissiva `anon all`. Tabela nova precisa de
`enable row level security` + policy explícita. RPC nova precisa de
`grant execute ... to anon` e `notify pgrst, 'reload schema'`.

## Marca

Dani é **feminina**. Copy, metatag, alt text e registro no banco usam
concordância feminina. Ela é a mascote de inglês adulto.

O cenário é rotativo por temporada (Halloween, Natal). Variante nova não
sobrescreve a anterior.

## Deploy

Nunca entregue arquivo para upload manual. Sempre via API do GitHub.

1. Valide o JS com `node --check` antes de commitar
2. `GET` do arquivo para pegar o `sha`
3. `PUT` em `contents` com `sha` + conteúdo em base64
4. Poll em `pages/builds/latest` até `built`
5. Confirme lendo via API com `Accept: application/vnd.github.raw`

## Regras

- Busque o arquivo atual antes de editar. Nunca escreva de memória.
- Confirme com o Pedro antes de qualquer operação destrutiva no banco.
- Sem travessão em texto de copy.
