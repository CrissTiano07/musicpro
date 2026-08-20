# CONTEXT · Órbita — Atendimento & OS (v2)

> Continuação do CONTEXT inicial. Este documento captura o estado após a longa sessão de
> maturação da **impressão / geração da OS**, incluindo a virada arquitetural mais importante
> do projeto até aqui. Foco no *porquê* de cada decisão, para servir de âncora ao retomar.

**Status:** em produção de teste, na nuvem, funcionando. OS e recibo saem como **PDF**.
**Arquivo:** `mvp-atendimento-os.html` (arquivo único, vanilla JS, sem build).
**Hospedagem:** GitHub Pages — `crisstiano07.github.io/supervisao/` (arquivo servido como `index.html`).

---

## 1. A VIRADA: de "imprimir HTML" para "gerar PDF"

Esta foi a decisão central da sessão, e vale entender o raciocínio para não regredir.

**O problema:** a OS precisa sair em **paisagem** (meia-folha, OS à esquerda + verso do QR à
direita). No notebook, o `window.print()` com `@page` em paisagem funcionava. **No iPhone, não** —
o WebKit do iOS **ignora `@page size/orientation`** e ainda aplica um autoajuste de escala próprio.
Tentativas de contornar (rotação por CSS transform, dobras) só pioraram (página extra, corte,
posicionamento errado). Rotação de impressão no iOS é um beco sem saída conhecido.

**A solução (pesquisada, não chutada):** parar de depender do motor de *impressão* do navegador
e passar a **gerar um PDF de verdade**, com tamanho e orientação **embutidos no arquivo**. Um PDF
paisagem é paisagem em qualquer aparelho — o iPhone não decide nada, só abre o arquivo pronto.

**Implementação:**
- Biblioteca `html2pdf.bundle` (jsPDF + html2canvas) via CDN (cdnjs).
- Função `gerarPDF(html, mode, nome)`: injeta o HTML em `#printDoc` posicionado **abaixo da tela**
  (`top:2600px; left:0` — NUNCA `left` negativo, que faz o html2canvas cortar a esquerda),
  espera as fontes (`document.fonts.ready`), captura com `html2canvas` (scale 2, para segurança
  de memória no mobile) e monta o PDF com `jsPDF` (`format:'a4'`, `orientation:` landscape p/ OS,
  portrait p/ recibo). Saída via `.save()`.
- **Bônus mobile-first:** como é PDF, o consultor **compartilha direto no WhatsApp** ou imprime.
  É o fluxo real de quem atende em domicílio.

**Trade-off aceito:** o PDF é uma "foto" (raster) do layout — texto não é selecionável, mas o
visual é preservado exatamente. Para um documento impresso, isso não importa. Se o texto sair
borrado em algum aparelho, o ajuste é aumentar o `scale` do html2canvas.

**⚠️ NÃO REGREDIR:** não voltar a `window.print()` para a OS achando que "simplifica". Ele quebra
no iOS. A geração de PDF é a decisão correta e testada.

## 2. Layout da OS (estado atual, aprovado)

Formato: **paisagem meia-folha**. OS ocupa a metade esquerda (~14,6 cm); metade direita é o
**verso reservado do QR de divulgação** (a "vitrine" do projeto). Linha de dobra tracejada entre elas.

Estrutura da face da OS, de cima para baixo:
- **Cabeçalho:** selo órbita + `ÓRBITA — [NOME DA ÓTICA]` (caixa-alta) à esquerda; à direita,
  `ORDEM DE SERVIÇO`, número grande (`OS-000003`), ref/data e **selo de status** (pílula preta,
  ex. "ENVIADO AO LAB."). Cabeçalho robusto: o número tem espaço reservado fixo à direita e o
  nome ocupa o resto sem sobrepor (corrige a sobreposição que aparecia no iOS).
- **Linha de identificação:** CNPJ + Tel/WhatsApp **do emitente** (não do cliente).
- **Bloco Cliente:** só o nome + célula "código QR" (discreta).
- **Receita:** tabela OD/OE × Esf./Cil./Eixo/Adição. Valores em mono, **negrito**, grade 1,5 px.
  Notação BR (vírgula, eixo com °, "plano"/"Sem correção" quando vazio).
- **Centragem:** tabela DNP / Altura / **DP total** (célula mesclada, soma automática).
- **Lente/Índice/Armação:** tabela ao lado da centragem.
- **Tratamentos:** checkboxes (só os marcados; ✓ em caixa com borda).
- **Observações:** caixa proporcional ao conteúdo.
- **Uso do laboratório:** faixa de 3 campos em branco (Controle interno, Recebido em, Previsão).
  Prazo é do laboratório preencher — o consultor NÃO imprime prazo.
- **Rodapé:** LGPD + duas assinaturas (cliente; carimbo do lab).

**Decisões de design travadas (não desfazer sem motivo forte):**
- **Sem fundos cinza** (zebra e preenchimentos) — o usuário reprovou; ficou só borda + peso de
  fonte. As únicas áreas preenchidas são as **barras pretas** (selo de status e cabeçalho do
  "Uso do laboratório"), que vieram do mockup aprovado.
- **`print-color-adjust: exact`** aplicado — necessário para as barras pretas saírem na impressão/PDF.
- **Rótulos em preto sólido** (nunca cinza-claro — some em impressora fraca).
- **Sem texto rotacionado** sobre o layout (o "corte/dobre" foi removido; a linha tracejada comunica).

## 3. LGPD e escopo — o que foi REJEITADO de propósito

Várias "análises de melhoria" (algumas geradas por IA) sugeriram adicionar campos. Foram
rejeitadas conscientemente, e é importante NÃO reintroduzi-las sem repensar:
- **Telefone / CPF do cliente na OS** → NÃO. Minimização LGPD: a OS vai ao laboratório, que não
  precisa de dado pessoal do cliente. (Um mockup chegou a mostrar o telefone do cliente — ignorar.)
- **Prazo prometido ao cliente** → NÃO. É critério do laboratório, não do consultor.
- **Valores (total/sinal/saldo) na OS** → NÃO. Valor é comercial, vai no **recibo** do cliente.
- **Campos técnicos de laboratório** (diâmetro, curva base, DIP, A/B/DBL, marca da lente,
  matriz de estações, rubrica por operador) → NÃO por ora. É documento *interno de laboratório
  industrial*, não a OS de um consultor domiciliar. Adicionar campo técnico é fricção no
  atendimento e deve vir de **dado real** de quais campos os laboratórios exigem, não de checklist.

**Princípio que emergiu:** a OS é a **vitrine do projeto** — cada peça que sai da mão do consultor
é primeira impressão. Isso eleva o padrão de qualidade de tudo, mas o inimigo nº 1 continua sendo
o **atrito no atendimento**: se digitar for mais lento que a caneta do consultor, ele volta ao papel.

## 4. Padronização de UX (entrada de dados)

Regra única aplicada em todo o app: **número clínico → modal-teclado; escolha → toque; texto →
teclado nativo.**
- **Receita e Medições** usam o **mesmo modal-teclado** (dígito = inteiro; passo ±0,25 dioptria /
  ±0,5 mm). Nenhum teclado nativo. Consistência = acessibilidade (pensando no consultor idoso).
- **Produto:** Tipo = chips (poucas opções); Material = lista empilhada (muitas opções); Tratamentos
  = chips múltiplos. Armação/Preço/Obs = texto (teclado nativo, correto para texto livre).
- **Barra de ação** (salvar / gerar PDF) se **recolhe durante a digitação** (classe `body.typing`),
  para o teclado não brigar por espaço.

## 5. Botão de saída — semiótica

O botão que gera o PDF diz **"Gerar PDF"** com **ícone de download** (seta pra baixo). Antes dizia
"Salvar e imprimir" com ícone de impressora — mentia sobre a ação (não imprime, gera arquivo).
Signo agora bate com a ação.

## 6. Háptica (resposta tátil)

Helper `haptic()` ligado nos toques principais (teclados, passos, abertura de célula, chips, botões).
- **Android:** `navigator.vibrate(9)` — funciona sólido.
- **iOS:** melhor-esforço via truque do `<input type="checkbox" switch>` (Safari 17.4+). A Apple
  **fechou a brecha no iOS 26.5**, então pode não vibrar nas versões mais novas. **Degrada sem
  quebrar** — se não vibra, o app funciona idêntico. Háptica garantida no iOS só viria com PWA/nativo.

## 7. Deploy — lições

- GitHub Pages teve um episódio de fila travada (`deployment_queued` / timeout) que **não era do
  código** — instabilidade da infra do GitHub; destravou sozinho com o tempo / novo commit. Plano B
  registrado: **Netlify** (arrastar o arquivo, sem fila), só precisaria adicionar o domínio nos
  Domínios Autorizados do Firebase.
- Anti-cache no teste: abrir em **aba anônima** com `?v=N` (incrementar N sempre).

## 8. Fios abertos (em ordem de retorno)

1. **Usar em atendimento real** — o passo de maior valor. O atrito de campo decide o resto.
2. **Encadeamento OD→OE** no modal de receita (ao confirmar OD, abrir OE).
3. **Scroll no notebook** (só rola pela barra, não pela roda). ⚠️ RESSALVA CRÍTICA: a correção
   NÃO pode reabrir o zoom/deslize acidental no mobile — são as mesmas regras (`touch-action`,
   `overscroll-behavior`, `user-select`). Separar por dispositivo e testar mobile + notebook lado a lado.
4. **API de cache do Firestore** — silenciar aviso `enableMultiTabIndexedDbPersistence`. ⚠️ risco de
   quebrar a persistência offline; só com teste do usuário confirmando que grava/lê e funciona offline.
5. **Extração da receita por foto** — a grande evolução. Câmera / imagem do WhatsApp → OCR
   client-side → extrai dados → **descarta a imagem** (minimização LGPD) ou comprime. Cristiano já
   tem experiência (extrator PDF client-side, MediaPipe). Planejar com rigor, incluindo o ângulo LGPD.
6. **QR de duplo propósito** — dado técnico pro laboratório (offline) + porta de divulgação
   (online). Espaço já reservado no verso. Bifurcação (um QR? dois? qual primeiro?) a resolver.
7. **Protocolo de recebimento digital** — evolução do status da OS (enviado → confecção → pronto →
   entregue) + futura confirmação do laboratório. No app, não no papel.

## 9. Dado de origem a corrigir (não é bug)

Em **Ajustes**, os campos do emitente devem conter os dados da ÓTICA: **Nome = "Super Visão"**
(não o nome pessoal) e **CNPJ do emitente** preenchido. Nos testes apareceu "ÓRBITA — CRISTIANO
MIRANDA" e "CNPJ —" porque esses campos estavam com dado pessoal / vazios. Os rótulos em Ajustes
já deixam claro que são dados do **emitente**, não do cliente.

## 10. Lembrete LGPD ⚠️ (segue de pé desde o v1)

A regra do Firestore (`auth.uid == uid`) protege o dado por consultor. Seguro para o teste solo.
Mas quando **outro consultor** atender **cliente real**, entra receita de terceiro (dado sensível):
valem as obrigações formais — termo de consentimento (o app captura), papéis Controlador (consultor)
/ Operador (plataforma). Não deixar para trás ao sair do teste solo.
