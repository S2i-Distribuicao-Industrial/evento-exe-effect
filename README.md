# Take it Easy — Landing page de inscrição

Página de confirmação de presença (RSVP) para o evento **Take it Easy**, da S2i Distribuição Industrial.
Site estático de uma página, com formulário que grava as inscrições em uma Google Planilha e
dispara e-mails automáticos de confirmação.

| | |
|---|---|
| **Evento** | 06 de outubro de 2026, começando às 18h |
| **Local** | Coco Bambu — Shopping Estação Cuiabá, Av. Miguel Sutil, 9300 - LUC 1001 |
| **Produção** | https://seffect.s2i.com.br (o antigo `takeiteasy.s2i.com.br` segue ativo: os e-mails já enviados têm link de cancelamento nele) |

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
├── cancelar.html       Página de cancelamento (seffect.s2i.com.br/cancelar?id=…)
├── Codigo.gs           Backend: inscrição, cancelamento e e-mails (NÃO versionado — ver abaixo)
├── Lembretes.gs        Backend: lembrete pré-evento + modos teste/produção (NÃO versionado)
├── src/
│   ├── IMAGENS/        Banner, fotos de eventos anteriores e do local
│   └── ICONS/          Ícones SVG e favicon
├── versoes_anteriores/ Rascunhos antigos (ignorados pelo git)
└── README.md
```

### Por que os `.gs` não estão no repositório

O `.gitignore` exclui `*.gs`. Os arquivos locais servem como backup e referência, mas
**a fonte da verdade é o editor do Apps Script**, onde eles rodam de fato. Eles
também contêm e-mails internos, que não devem ficar expostos no repositório.

Os dois arquivos vivem **no mesmo projeto do Apps Script** e compartilham o escopo global
(`Lembretes.gs` usa o `CONFIG` e as funções do `Codigo.gs`).

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

### Cancelamento de inscrição

Os e-mails de confirmação e de lembrete trazem um botão vermelho **"Cancelar minha inscrição"**,
que aponta para `https://seffect.s2i.com.br/cancelar?id=<UUID da inscrição>`
(`CONFIG.URL_CANCELAMENTO`). A página `cancelar.html` é estática e conversa com o Apps Script
pelo mesmo `SCRIPT_URL` do formulário — se a implantação mudar, atualize os dois arquivos.

O fluxo tem **duas etapas de propósito**:

1. Abrir o link só **consulta** (`GET ?acao=consultar&id=…`, que devolve apenas o primeiro nome).
   Nada é apagado.
2. O botão **Confirmar cancelamento** envia `POST {acao: "cancelar", id}`, que chama
   `cancelarInscricao(id)`.

Links antigos no formato `<URL do Web App>?acao=cancelar&id=…` (e-mails de confirmação enviados
antes da `cancelar.html`) continuam funcionando: abrem a página servida pelo próprio Web App.

A separação existe porque clientes de e-mail e antivírus **abrem os links das mensagens**
para inspecioná-los. Um GET que apagasse dados direto seria disparado por esses robôs, e a
pessoa perderia a inscrição sem ter clicado em nada.

O **UUID da coluna `ID` é a credencial**. Só quem recebeu o e-mail o conhece, e ele é
impossível de adivinhar. É por isso que o cancelamento nunca aceita e-mail como
identificador: qualquer um poderia cancelar a inscrição alheia sabendo só o endereço.

Ao confirmar, o script **não apaga nada**: marca como `Cancelado` a linha daquele ID e as
demais linhas `Ativa` com o mesmo e-mail (duplicados). Linhas `Substituída` ficam como estão.
Os dados continuam na planilha para consulta futura. Em seguida, avisa o marketing
(`EMAIL_MARKETING`, ver *Contagem regressiva por e-mail*). O aviso traz nome, e-mail, telefone,
empresa e quantos participantes confirmados restaram, para o marketing saber quando chamar
alguém da lista de espera. Sai da conta que executa o Web App.

Status possíveis na coluna `Status`:

| Status | Significado | Recebe lembretes? |
|---|---|---|
| `Ativa` | Inscrição válida | Sim |
| `Substituída` | A pessoa se inscreveu de novo com o mesmo e-mail; vale a linha mais nova | Não |
| `Cancelado` | A pessoa cancelou pelo link do e-mail | Não |

Um link já usado mostra "Inscrição não encontrada" e não cancela duas vezes. Se a pessoa se
inscrever de novo depois de cancelar, ganha uma linha `Ativa` nova e a `Cancelado` fica como
histórico.

### Publicar alterações do backend

**Implantar → Gerenciar implantações → ✏️ (editar) → Versão: Nova versão → Implantar**

Use sempre a implantação **existente**. "Nova implantação" gera uma URL `/exec` diferente
e quebra o formulário até você atualizar o `SCRIPT_URL` no `index.html`.

---

## Contagem regressiva por e-mail

`Lembretes.gs` envia aos inscritos uma contagem regressiva, sempre com o botão **Cancelar minha
inscrição** (o mesmo fluxo de duas etapas descrito acima). Os dias ficam em
`CONFIG.DIAS_LEMBRETE`:

| Dias antes | Data | Assunto |
|---|---|---|
| 10 | 26/09 | Lembrete S2i Effect — faltam 10 dias |
| 7 → 2 | 29/09 → 04/10 | Lembrete S2i Effect — faltam N dias |
| 1 | 05/10 | Lembrete S2i Effect — falta 1 dia |
| 0 | 06/10 | S2i Effect — É hoje! (data, horário e local) |

Com a agenda ativa, um acionador diário roda `lembreteAutomatico` entre 9h e 10h (Cuiabá),
confere a data e envia se for dia de lembrete. Depois do evento, ele se remove sozinho.

Com o `Lembretes.gs` aberto no editor, escolha a função no menu ao lado de **Executar**:

| Função | O que faz |
|---|---|
| `teste1_enviarParaMeuEmail` | Lembrete de hoje só para `EMAIL_TESTE_INTERNO` |
| `teste2_enviarParaEmailPessoal` | Lembrete de hoje só para `EMAIL_TESTE_EXTERNO` |
| `teste3_verEmailDoDiaDoEvento` | Prévia do "É hoje!" para `EMAIL_TESTE_INTERNO` |
| `envio1_simularHoje` | Mostra no log o calendário e quem receberia hoje (não envia) |
| `envio2_enviarLembreteDeHoje` | Envia agora o lembrete de hoje, se hoje for dia de lembrete |
| `agenda1_ativarContagemAutomatica` | Liga o envio automático diário |
| `agenda2_desativarContagemAutomatica` | Desliga o envio automático |

Os e-mails saem da conta que executa — no automático, da conta que ativou a agenda.

Cada pessoa recebe cada lembrete uma vez só: a coluna **"Lembretes enviados"** guarda quais já
foram (ex.: `D-10 · D-7 · D-6`). O prefixo `D-` é proposital: gravar `10, 7` faria o Sheets em
português ler `10,7` como número decimal.

Os testes criam uma linha `TESTE Lembrete` na planilha, então o cancelamento é real: ao
confirmar, a linha some de fato. O aviso de cancelamento vai para `EMAIL_MARKETING`, exceto
quando a linha cancelada é de teste (nome começando com `TESTE`), que avisa só o
`EMAIL_TESTE_INTERNO` com assunto `[TESTE]`.

### Proteções

- **Sem reenvio.** Cada lembrete é registrado na coluna `Lembretes enviados` (criada sozinha).
  Rodar de novo — ou rodar na mão no mesmo dia do automático — só alcança quem ainda não
  recebeu aquele lembrete. Também serve para retomar uma execução interrompida.
- **Quem cancela para de receber**: a linha vira `Cancelado`, e só linhas `Ativa` recebem.
  O status é conferido de novo **logo antes de cada e-mail**, então quem cancela enquanto o
  envio do dia está em andamento também não recebe.
- **Cancelar durante o envio funciona.** O envio não segura a trava do script enquanto os
  e-mails saem (usa uma marca própria de "envio em andamento"), então cancelamentos e
  inscrições entre 9h e 10h não recebem "servidor ocupado".
- **Só `Ativa`, um por e-mail, sem testes.** Linhas `Substituída`, e-mails duplicados e linhas
  de teste esquecidas (`TESTE…`, `LINHA DE DIAGNOSTICO`, `@example.com`) são ignorados.
- **Cota.** Aborta antes de enviar se `MailApp.getRemainingDailyQuota()` for menor que a lista
  (100/dia em conta Gmail comum, 1.500 em Workspace).
- **E-mail de teste que é de inscrito real** aborta a execução: o cancelamento de teste
  cancelaria a inscrição dele.
- **Depois do evento** nada é enviado, e o acionador se remove sozinho.
- **Nada falha em silêncio.** Todo impedimento aparece como erro vermelho no log do editor.

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

**Telefone validado nas duas pontas.** `type="tel"` não valida nada — o navegador aceita
letras. O `index.html` aplica uma máscara (só dígitos, formato `(65) 99999-9999`) e um
`pattern` que bloqueia o envio incompleto; o backend repete a checagem em `normalizarTelefone()`,
recusa o que não for um telefone com DDD (10 ou 11 dígitos, `+55` opcional) e grava sempre no
mesmo formato. A validação do servidor é a que vale: a URL do script é pública e pode receber
POST sem passar pelo formulário.

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

Os domínios `seffect.s2i.com.br` e `takeiteasy.s2i.com.br` são *custom domains* do projeto (Worker `evento-exe-effect`, em Domains & Routes). Não remova o antigo: os e-mails já enviados apontam para ele. Como a zona
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
      como "S2i Effect". Os e-mails saem com nome diferente do site.
- [ ] **Peso das fotos**: as fotos de eventos e do local em `src/IMAGENS/` ainda somam
      ~8 MB, com PNGs de até 2 MB. Converter para JPEG q90 reduziria bastante.
- [x] **Banner e imagem de compartilhamento** — resolvido. Ver seção *Imagens* abaixo.

## Imagens

O banner tem três arquivos, com papéis distintos:

| Arquivo | Uso | Tamanho |
|---|---|---|
| `takeeasy.png` | **Mestre.** Original sem perdas, não usado pela página | 1,6 MB |
| `takeeasy.jpg` | Banner exibido no site | 276 KB |
| `og-takeeasy.jpg` | Prévia ao compartilhar o link (1200×675) | 173 KB |

Ao regerar, **sempre parta do `takeeasy.png`**. Recomprimir um JPEG já comprimido acumula
perda de geração — o erro do passo anterior é tratado como detalhe legítimo e preservado.

Parâmetros usados: `quality=90`, `subsampling=0`, `optimize=True`, `progressive=True`.

O `subsampling=0` (4:4:4) é o ponto crítico. Compressores usam 4:2:0 por padrão, que
descarta 3/4 da informação de cor — imperceptível em fotografia, mas destrutivo em texto
e áreas de cor chapada, que é exatamente do que um convite gráfico é feito. Desligar custa
~20% de tamanho e reduz o erro em 35%.
- [ ] **Tailwind via CDN**: a build de desenvolvimento é desaconselhada em produção pelo
      próprio projeto. Gerar o CSS e servir localmente.
- [ ] **Limpar linhas de teste** da aba `Inscrições` antes da divulgação.
