# 🧮 Math Kids

Jogo de matemática para crianças de **7 a 10 anos**: soma, subtração, multiplicação
e divisão em fases curtas, com login por **avatar + PIN**, progresso salvo na nuvem
e um mini-reconhecedor de dígitos desenhados à mão (a criança escreve a resposta
com o dedo na tela).

Site **100% estático** (HTML + CSS + JavaScript puro, sem build) com backend no
**Supabase**. Hospedado na **Vercel**.

---

## ✨ Como funciona

| Tela | Arquivo | O quê |
|---|---|---|
| **Quem vai jogar** | `index.html` + `js/script.js` | grade de avatares (vem do Supabase), teclado de PIN, e a "Área do responsável" (cadastro da criança + consentimento) |
| **Mapa de fases** | `jogar.html` + `js/jogar.js` | fases liberadas/bloqueadas, estrelas e melhor pontuação por fase |
| **Partida** | `partida.html` + `js/partida.js` | as contas da fase, vidas, cronômetro opcional e a lousa onde a criança **desenha o resultado** |

### Login e segurança
- O PIN (4 dígitos) **nunca** trafega nem é comparado no navegador. O hash roda
  dentro do Postgres, nas funções RPC do Supabase.
- O login devolve um **token de sessão opaco** (hex, expira em 12 h), guardado só
  no `sessionStorage`. Todo salvamento de progresso exige esse token.
- RLS ligado em todas as tabelas **sem policies**: a chave pública só consegue
  *executar* as funções RPC, nunca ler/escrever direto nas tabelas.

### Fases
`js/fases.js` descreve só os parâmetros de dificuldade de cada fase; `js/gerador.js`
monta as contas garantindo: subtração nunca negativa, divisão sempre exata,
resposta entre 0 e 99 e sem conta repetida na mesma fase. São 10 fases iniciais
(`id: 1..10`) mais fases extras (`id: 101..103`).

### Reconhecimento de dígitos
`js/reconhecimento.js` roda uma MLP `784 → 128 → 10` **em JavaScript puro**, com os
pesos quantizados em int8 dentro de `js/modelo-mnist.json` (~135 KB, sem
TensorFlow.js nem WASM). O modelo é treinado offline pelos scripts em
`ferramentas/` (ver abaixo).

---

## 🗂️ Estrutura

```
math-kids/
├── index.html, jogar.html, partida.html   páginas do jogo
├── css/
│   ├── style.css                          base / telas de menu
│   └── jogo.css                            tela de partida
├── js/
│   ├── script.js                          tela inicial (avatares, PIN, cadastro)
│   ├── jogar.js                           mapa de fases
│   ├── partida.js                         loop de uma partida
│   ├── fases.js / gerador.js              definição das fases e geração das contas
│   ├── avatares.js                        SVGs dos rostos dos avatares
│   ├── progresso.js / sessao.js           progresso na nuvem + token de sessão
│   ├── reconhecimento.js                  inferência da MLP (JS puro)
│   ├── modelo-mnist.json                  pesos int8 do modelo (gerado)
│   ├── supabaseClient.js / config.js      cliente Supabase + credenciais
├── supabase/
│   ├── schema.sql                         tabelas + funções RPC (rodar no SQL Editor)
│   └── LEIA-ME.md                         passo a passo da integração
├── ferramentas/                           scripts de treino (Python) — NÃO vão pro deploy
├── vercel.json                            headers de cache
└── .vercelignore                          o que fica de fora do deploy
```

---

## 🚀 Rodar localmente

Precisa servir por HTTP (os `import` de ES modules não funcionam via `file://`).

```bash
# a partir da pasta math-kids/
npx serve .
# ou, com Python:  python -m http.server 5500
```

Abra `http://localhost:3000` (ou a porta indicada). No VS Code, a extensão
**Live Server** também serve.

---

## 🔌 Configurar o Supabase

Passo a passo completo em [`supabase/LEIA-ME.md`](supabase/LEIA-ME.md). Resumo:

1. Crie um projeto em [supabase.com](https://supabase.com).
2. **SQL Editor** → cole `supabase/schema.sql` → execute (cria tabelas, popula os
   avatares e cria as funções RPC). Re-executar é seguro (`if not exists` /
   `create or replace`).
3. *Project Settings → API*: copie a **Project URL** e a **anon/publishable key**.
4. Preencha `js/config.js` com esses dois valores.

Não precisa de variável de ambiente, CORS nem redirect URL — o app usa só
`supabase.rpc(...)`.

---

## 🧠 Re-treinar o modelo de dígitos

```bash
cd ferramentas
python -m pip install numpy
python treinar_modelo.py        # baixa o MNIST (~11 MB) p/ .cache/ e gera js/modelo-mnist.json
python gerar_amostras_teste.py  # opcional: gera amostras-teste.json p/ testar-reconhecimento.html
```

`ferramentas/testar-reconhecimento.html` mede a taxa de acerto no próprio
navegador (critério de aceite: ≥ 80%; o modelo atual fica em ~97–98%).

---

## ☁️ Deploy na Vercel

Site estático, sem build. **Root Directory** = esta pasta (a que tem
`index.html` + `vercel.json`); Framework Preset = **Other**; sem Build Command.

```bash
npm i -g vercel
vercel          # primeiro deploy (cria o projeto)
vercel --prod   # publica em produção
```

`vercel.json` define o cache (modelo imutável por 1 ano; JS/CSS revalidados a cada
deploy; HTML sem cache). `.vercelignore` mantém `ferramentas/` e `supabase/` fora
do site publicado.

---

## ⚠️ Limitações conhecidas

- **PIN de 4 dígitos sem rate-limit no servidor** — força bruta online é viável;
  o alvo é uso escolar de baixo risco (as RPCs não expõem dados pessoais).
- **`supabase-js` carregado de `esm.sh`** sem versão fixa — rede que bloqueia CDNs
  (comum em escolas) derruba o app. Ideal: vendorizar a lib e pinar a versão.
- **Recuperar PIN esquecido** (história 3.6): hoje só mostra um `alert()`.
