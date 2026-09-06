# Take it Easy — Landing page de inscrição

Página de confirmação de presença (RSVP) para o evento **Take it Easy**, da S2i Distribuição Industrial.
Site estático de uma página, com formulário que grava as inscrições em uma Google Planilha e
dispara e-mails automáticos de confirmação.

| | |
|---|---|
| **Evento** | 06 de outubro de 2026, das 18h às 21h |
| **Local** | Coco Bambu — Shopping Estação Cuiabá, Av. Miguel Sutil, 9300 - LUC 1001 |
| **Produção** | https://takeiteasy.s2i.com.br |

---

## Como funciona

```
  Visitante
     │  preenche o formulário
     ▼
  index.html  ──── fetch POST (JSON) ────►  Google Apps Script (Web App)
  (site estático)                                    │
                                                     ├──► grava linha na Google Planilha
                                                     ├──► e-mail de confirmação ao participante
                                                     └──► e-mail de aviso a vendas@s2i.com.br
```

Não há servidor próprio nem banco de dados. O backend inteiro é um script do Google Apps Script
publicado como *Web App*, e a "base de dados" é uma aba de planilha.

---

## Estrutura

```
.
├── index.html          Página completa: HTML, CSS (Tailwind CDN) e JS inline
├── Codigo.gs           Cópia local do backend (NÃO versionado — ver abaixo)
├── src/
│   ├── IMAGENS/        Banner, fotos de eventos anteriores e do local
│   └── ICONS/          Ícones SVG e favicon
├── versoes_anteriores/ Rascunhos antigos (ignorados pelo git)
└── README.md
```

### Por que o `Codigo.gs` não está no repositório

O `.gitignore` exclui `*.gs`. O arquivo local serve como backup e referência, mas
**a fonte da verdade é o editor do Apps Script**, onde ele roda de fato. O arquivo
também contém o e-mail do organizador, que não deve ficar exposto no repositório.

Ao alterar o backend, altere no editor do Apps Script — e, se quiser manter o backup
em dia, replique no arquivo local.

---

## Backend — Google Apps Script

O script está vinculado à planilha (**Extensões → Apps Script**) e grava na aba `Inscrições`,
criada automaticamente na primeira execução com o cabeçalho:

`ID | Data/Hora | Status | Nome | E-mail | Telefone | Empresa | Cargo | Como conheceu | Observações`

### Configuração

Todas as constantes ficam no objeto `CONFIG`, no topo do arquivo:

| Constante | Descrição |
|---|---|
| `ID_PLANILHA` | ID extraído da URL da planilha |
| `NOME_ABA` | Aba de destino (`Inscrições`) |
| `EMAIL_REPRESENTANTE` | Recebe o aviso de cada nova inscrição |
| `NOME_EVENTO` | Alimenta o assunto e o cabeçalho dos e-mails |
| `DATA_EVENTO` / `LOCAL_EVENTO` | Exibidos no e-mail de confirmação |
| `URL_LANDING_PAGE` | Link do botão no e-mail |
| `LIMITE_EMAILS_POR_HORA` | Teto de envios automáticos (padrão: 30) |
| `CAMPOS` | Define colunas, rótulos e obrigatoriedade |

> As `chave` de `CAMPOS` precisam bater **exatamente** com os `id` dos campos no `index.html`.
> Se renomear um campo em um lado, renomeie no outro — senão a coluna chega vazia, sem erro visível.

### Publicar alterações do backend

**Implantar → Gerenciar implantações → ✏️ (editar) → Versão: Nova versão → Implantar**

Use sempre a implantação **existente**. "Nova implantação" gera uma URL `/exec` diferente
e quebra o formulário até você atualizar o `SCRIPT_URL` no `index.html`.

---

## Decisões de segurança

O endpoint do Apps Script é público por natureza — a URL fica visível no código-fonte da
página. As proteções abaixo partem desse pressuposto.

**Injeção de fórmula no Sheets.** Valores começando com `=`, `+`, `-` ou `@` viram fórmula
ao serem gravados. Uma fórmula como `IMPORTXML` conseguiria enviar o conteúdo das demais
células para um servidor externo assim que alguém abrisse a planilha. `sanitizarParaPlanilha()`
prefixa esses valores com apóstrofo, forçando texto puro.

**Sem sobrescrita de registros.** Inscrição repetida com o mesmo e-mail gera uma linha nova
e marca a anterior como `Substituída`. Permitir *update* por e-mail deixaria qualquer pessoa
apagar os dados de um inscrito conhecendo apenas o endereço dele.

**Limite de envios.** Contador horário em `PropertiesService` protege a cota do Gmail
(100 e-mails/dia em conta comum) contra flood. Se o teto for atingido, **a inscrição ainda
é gravada** — apenas os e-mails são pulados. O dado nunca se perde por causa do envio.

**Honeypot.** Campo `website`, invisível para pessoas. Se vier preenchido, a requisição é
descartada silenciosamente (responde sucesso para não ensinar o robô).

**`doGet` não devolve dados.** Só responde um *health-check*. Como a URL é pública, expor
a lista de inscritos ali seria vazamento direto de dados pessoais.

**Escape de HTML** nos corpos dos e-mails, e o detalhe técnico de erros fica só no log
interno, nunca na resposta ao cliente.

**LGPD.** O formulário exige consentimento explícito. A lista de participantes contém dados
pessoais e vive apenas na planilha — o `.gitignore` bloqueia `*.csv`, `*.xlsx` e afins para
evitar que uma exportação entre no repositório por descuido.

---

## Deploy

O site é publicado via **Cloudflare Pages**, conectado a este repositório: todo push na
branch `main` dispara um novo deploy automaticamente.

O domínio `takeiteasy.s2i.com.br` é um *custom domain* do projeto no Pages. Como a zona
`s2i.com.br` está na mesma conta Cloudflare, o registro DNS e o certificado SSL são
criados automaticamente.

Não é necessário build: é HTML estático (*framework preset* `None`, *build command* vazio,
*output directory* `/`).

---

## Armadilhas conhecidas

Problemas que já custaram tempo neste projeto:

**Implantar como Biblioteca em vez de App da Web.** Gera uma URL `/macros/library/d/...`,
que não responde a POST nem envia cabeçalho CORS. Sintoma: `Failed to fetch` no console.
A URL correta contém `/macros/s/` e termina em `/exec`.

**Executar a função errada no editor.** O seletor vem em `doGet` por padrão. Tanto `doGet`
quanto `doPost` rodados manualmente concluem com sucesso e não gravam nada — parece que
"não aconteceu nada". Para testar, use `testarEnvio`.

**Esquecer de publicar nova versão.** Alterar o código não altera o que está no ar. A
implantação serve uma versão congelada até você publicar uma nova.

**Alterar a URL do endpoint.** O `SCRIPT_URL` no `index.html` precisa acompanhar qualquer
mudança de implantação.

---

## Pendências

- [ ] **Nome divergente**: a página exibe "Take it Easy" e o `NOME_EVENTO` do backend está
      como "The EasyEffect". Os e-mails saem com nome diferente do site.
- [ ] **Peso das imagens**: `src/IMAGENS/` soma ~9,5 MB, com PNGs de até 2 MB. Converter
      para JPEG/WebP a 1600px reduziria a página para menos de 1,5 MB.
- [ ] **Imagem de compartilhamento**: `takeeasy.png` tem 1,6 MB. O WhatsApp costuma ignorar
      prévias acima de ~300 KB. Gerar um `og-image.jpg` de 1200×630 abaixo desse limite.
- [ ] **Tailwind via CDN**: a build de desenvolvimento é desaconselhada em produção pelo
      próprio projeto. Gerar o CSS e servir localmente.
- [ ] **Limpar linhas de teste** da aba `Inscrições` antes da divulgação.
