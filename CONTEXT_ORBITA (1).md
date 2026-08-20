# CONTEXT · Órbita — Atendimento & OS óptico domiciliar

> Ferramenta de trabalho para o **consultor óptico autônomo (MEI)** que atende em domicílio:
> registra receita, mede, escolhe produto e gera a Ordem de Serviço para o laboratório.
> Este é o **Motor 1** do produto (uso diário, gancho de hábito). O Motor 2 (compra
> coletiva) está deliberadamente fora deste MVP.

**Status:** base em produção de teste, na nuvem, funcionando. Testado solo pelo desenvolvedor.
**Stack:** um único arquivo HTML, vanilla JS, sem build. Firebase (Firestore + Auth).
**Hospedagem atual:** GitHub Pages (`crisstiano07.github.io/supervisao/`).

---

## 1. Arquitetura

Aplicativo de página única, sem framework e sem etapa de build — decisão consciente para
manter a manutenção baixa e coerente com o stack do desenvolvedor. A "seriedade" da base
não está em tooling, e sim em três coisas: **camada de dados isolada, modelo por documento
e segurança por usuário.**

- **Estado em memória:** um único objeto `S` (`{config, clientes[], ordens[], counters}`)
  é a fonte única de verdade. A interface lê e escreve nele.
- **Camada de dados (repositório):** o *único* ponto que conhece o Firestore são cinco
  funções — `repoLoadAll`, `saveConsultor`, `saveCliente`, `saveOrdemDoc`, `deleteOrdemDoc`.
  A interface nunca fala com o banco direto. Trocar de backend no futuro = mexer só aqui.
- **Persistência offline** ativada (visitas com sinal ruim gravam local e sincronizam depois).

## 2. Modelo de dados (Firestore)

Namespaced por consultor autenticado, desde o dia um:

```
consultores/{uid}                    → { config:{nome,cnpj}, counters:{local} }
consultores/{uid}/clientes/{id}      → um documento por cliente
consultores/{uid}/ordens/{id}        → um documento por OS
```

Cada OS/cliente é um documento próprio, escrito individualmente (nada de reescrever o estado
inteiro; sem "último a salvar vence"). O ID do documento garante unicidade.

## 3. Segurança e infraestrutura

- **Projeto Firebase:** `orbita-b2b55`, separado do NIT (bancos, auth e cota isolados).
- **Firestore região:** `southamerica-east1` (São Paulo) — residência do dado no Brasil.
- **Auth:** Google (login por popup). Tela de login como portão antes do app.
- **Regras publicadas:**
  ```
  match /consultores/{uid}/{document=**} {
    allow read, write: if request.auth != null && request.auth.uid == uid;
  }
  ```
  Cada consultor só acessa o próprio galho.
- **Domínio autorizado:** `crisstiano07.github.io` (raiz cobre qualquer repositório).

## 4. Funcionalidades entregues

- **Fluxo de atendimento** em uma tela: cliente, receita, medições, produto, consentimento.
- **Receita** por olho (OD/OE) com **modal de digitação dedicado** (teclado de dioptria,
  passo ±0,25, um olho por vez), validação de faixa (eixo 0–180, dioptrias, adição).
- **Medições** com DNP e altura por olho e **DP total** somado automaticamente.
- **Numeração** sequencial por consultor (`OS-000047` = sua `#47`).
- **Histórico do cliente:** ao reatender, recupera a última receita com um toque.
- **Transposição** de eixo/cilíndrico (notação ±).
- **Impressão** de OS e recibo dimensionada para o porta-OS (23 cm), com **telefone e
  endereço fora da impressão** (minimização — ficam só no sistema).
- **Consentimento LGPD** obrigatório para salvar, com carimbo por OS.

## 5. Decisões e porquês

- **Firestore, não Realtime Database:** o RTDB não tem região no Brasil; o Firestore tem
  São Paulo. Além disso, o modelo por documento do Firestore já é a arquitetura correta
  (por registro, seguro por usuário) sem custo extra.
- **Número único por consultor:** a numeração dupla (global + local) existia só para evitar
  colisão no banco; o ID de documento do Firestore resolve isso, então colapsou em um número.
- **Arquivo único, sem framework:** correto para um produto de dev solo nesta fase.
  Separar em framework agora seria retrabalho disfarçado de progresso.
- **Modelo de monetização (definido, não implementado):** SaaS de gestão + vitrine de
  intenção — baixo risco jurídico (não é corretagem, não é intermediário ativo). No MVP,
  o consultor paga uma assinatura simbólica pelo Motor 1; a monetização pelo fornecedor
  (inteligência de demanda) vem quando houver grupo. Comissão sobre venda está fora por
  princípio (corretagem + fere o valor de "consultor mais beneficiado").
- **Papéis LGPD:** o consultor (MEI) é **Controlador**; a plataforma é **Operador**.
  Minimização aplicada (telefone/endereço não vão ao laboratório).

## 6. Experimento de validação (90 dias)

O MVP existe para responder duas perguntas, não para lucrar:
- **Sinal 1 — hábito:** o consultor usa o Motor 1 espontaneamente? (≥1 OS/semana em 3 de 4).
- **Sinal 2 — disposição a pagar:** mantém a assinatura após o período grátis?
- **Portão:** ambos positivos → construir o Motor 2 sobre base real. Só o Sinal 1 → útil,
  monetizar pelo fornecedor. Nenhum → parar/pivotar.

## 7. Limite de LGPD ainda aberto ⚠️

A arquitetura está pronta, mas a fronteira jurídica continua de pé: **no momento em que
outro consultor atender um cliente real, entra receita de terceiro (dado sensível de saúde).**
Aí valem as obrigações — termo de consentimento (o app já captura), papéis Controlador/Operador
formalizados. O que falta não é técnico; é o combinado com quem for usar. Não deixar isso
para trás ao sair do teste solo.

## 8. Próximos passos mapeados (não urgentes)

- Usar em atendimentos reais e deixar o Sinal 1 falar (é o dado que decide o resto).
- "Confirmar OD → abrir OE" encadeado no modal de receita.
- Sincronização em tempo real entre abas/aparelhos (listener `onSnapshot`, com cuidado
  para não sobrescrever formulário em edição).
- Escrita por nó já é feita; falta transação no contador se virar multi-consultor.
- v2 de segurança já está essencialmente feita (namespaced + Auth); reforçar antes do dado real.
