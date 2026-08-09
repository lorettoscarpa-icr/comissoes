# CRM WhatsApp — Loretto Scarpa

Painel de CRM das quatro vendedoras (**Jhennifer, Ialey, Michelly, Natália**), num arquivo
único: `crm.html`. Mesmo Firebase da Central de Pagamentos (`loretto-scarpa-l-icr`), mesmo
login, mesma identidade visual. A Central não foi tocada.

Abrir: `crm.html` no navegador. Quem já estiver logado na Central entra direto.

---

## O que o painel mostra

| Aba | Para quê |
|---|---|
| **Painel** | Mensagens recebidas por vendedora, conversão de cada uma, origem dos leads (anúncio / Instagram / orgânico / indicação), funil de atendimento e qual canal traz cliente que compra. |
| **Leads** | Todos os clientes, filtráveis por vendedora, origem, etapa e período. Clique abre a ficha para editar. Exporta CSV. |
| **Repescagem** | Cinco filas prontas, com o texto da mensagem já sugerido e link direto pro WhatsApp. |
| **Agenda** | Quem combinou de vir à loja — atrasados, hoje, amanhã, próximos dias. |
| **Importar** | Onde o Claude entrega a varredura do WhatsApp. |

---

## Onde os dados ficam

Duas coleções novas no Firestore. Nenhuma delas encosta nas coleções da Central
(`fechamentos`, `metas`, `atingimentos`, `horasExtras`, `equipeFixa`, `adiantamentos`,
`pagamentos`).

```
crm_leads/{telefone}     um documento por cliente — o telefone é a chave
crm_msgs/{AAAA-MM-DD}    volume diário de mensagens por vendedora
```

> **Antes do primeiro uso:** conferir se as regras de segurança do Firestore liberam
> leitura e escrita de `crm_leads` e `crm_msgs` para usuário autenticado. Se as regras
> listarem coleção por coleção, as duas precisam ser adicionadas, senão o painel abre
> vazio e a importação falha.

---

## Como o Claude alimenta o CRM

O Claude varre o WhatsApp Web de uma vendedora usando o skill
`loretto-scarpa-atendimento` (extensão Claude in Chrome), classifica cada conversa e
entrega o resultado de uma destas duas formas:

1. **Colando o JSON** na aba *Importar* e clicando em Importar.
2. **Chamando a API** no console da aba do `crm.html`:
   ```js
   await window.CRM.importar({ vendedora: 'Jhennifer', leads: [ ... ] })
   ```

### Formato

```json
{
  "vendedora": "Jhennifer",
  "mensagensDia": { "2026-08-09": { "recebidas": 47, "enviadas": 39 } },
  "leads": [{
    "telefone":        "62991234567",
    "nome":            "Danilo",
    "origem":          "anuncio | instagram | organico | indicacao | desconhecida",
    "campanha":        "LOAFER DUBAI",
    "modelo":          "Loafer Dubai",
    "cor":             "Preto",
    "numeracao":       "41",
    "etapa":           "novo | atendido | preco | numeracao | localizacao | agendado | compareceu | vendido | perdido",
    "tags":            ["preco_so_audio", "sem_resposta"],
    "primeiroContato": "2026-08-03",
    "ultimaCliente":   "2026-08-05",
    "ultimaLoja":      "2026-08-05",
    "msgsCliente":     12,
    "msgsLoja":        9,
    "agendadoPara":    "2026-08-12",
    "agendadoHora":    "15:00",
    "compareceu":      true,
    "valorVenda":      349.90,
    "obs":             "texto livre"
  }]
}
```

Só `telefone` é obrigatório. **Mande apenas o que você conseguiu ler da conversa** — o
resto fica como está.

### Regras de gravação

- O telefone é a chave e é normalizado (`(62) 99123-4567`, `+55 62 99123 4567` e
  `62991234567` viram o mesmo documento). O mesmo cliente nunca duplica.
- Campo ausente **não apaga** o que já está salvo.
- `tags` **soma** às que já existem. A edição manual no painel substitui.
- `etapa` **nunca anda para trás** numa importação — só `perdido` pode. O painel guarda
  à parte a etapa mais avançada já alcançada, para o funil não encolher quando alguém é
  marcado como perdido.
- `msgsCliente` / `msgsLoja` ficam com o **maior** valor entre o salvo e o enviado.
- `repescagens` são deduplicadas por data — reimportar a mesma varredura não infla o
  contador nem faz o cliente estourar o teto por engano.

### Outras chamadas disponíveis

```js
window.CRM.resumo()        // métricas do mês selecionado
window.CRM.filas()         // filas de repescagem já com o texto sugerido
window.CRM.leads()         // todos os leads
window.CRM.lead('62991234567')
window.CRM.recarregar()    // relê o Firestore
window.CRM.vocabulario     // valores válidos de vendedora, origem, etapa e tags
window.CRM.classificar('Olá, vim no Instagram e quero saber mais')  // testa uma mensagem
window.CRM.regras()        // as regras de origem e o que cada uma reconhece
window.CRM.lembretes()     // filas de visita marcada, com o texto pronto
```

### O que a regra resolve e o que precisa da IA

| Campo | Quem decide |
|---|---|
| `origem` | **Regra de texto** — template de anúncio é exato |
| `etapa` | IA — até onde a conversa chegou |
| `tags` | IA — preço só em áudio, sem resposta, pediu desconto, já comprou |
| `modelo` · `cor` · `numeracao` | IA — o cliente escreve de mil formas |
| `agendadoPara` | IA — "sábado", "depois do dia 20" viram data |
| `compareceu` | IA — se apareceu na loja |

---

## Classificação automática de origem

A origem do lead é lida da **primeira mensagem do cliente**, por regra de texto — não por
alguém marcar nada, e sem IA.

O motivo é que as mensagens de anúncio do Meta são **template fixo**: chegam sempre com o
mesmo texto. Casar padrão nisso é exato, custa zero e nunca inventa. IA num problema
desses só adicionaria erro.

| Regra | Reconhece | Vira |
|---|---|---|
| `anuncio_interesse` | "quero / tenho interesse em saber mais informações" | Anúncio |
| `anuncio_cidade` | "sou de Goiânia e quero saber mais" | Anúncio |
| `anuncio_instagram` | "vim no Instagram e quero saber mais" | Anúncio |
| `anuncio_modelo_caps` | modelo escrito em MAIÚSCULAS, como no banner | Anúncio |
| `instagram_bio` | "vim do link da bio" | Instagram orgânico |
| `instagram_perfil` | citou perfil, story, reels ou "achei vocês no insta" | Instagram orgânico |
| `indicacao` | "me indicou", "meu amigo comprou aí" | Indicação |

São testadas de cima para baixo e a primeira que casar vence. **A ordem importa:** o
template do anúncio veiculado no Instagram é testado *antes* das regras de Instagram
orgânico — senão todo lead de anúncio do Instagram viraria orgânico.

Três garantias:

- **Origem definida na mão nunca é sobrescrita.** Mexeu na ficha, trava.
- **O que nenhuma regra classificou cai na fila de conferência** (aba Automações), junto
  com o que casou só com confiança média. A automação não erra em silêncio.
- **`Reclassificar tudo`** roda as regras sobre a base inteira, então padrão novo vale
  também para o histórico — não só daqui pra frente.

Para adicionar um padrão: inclua a frase em `REGRAS_ORIGEM`, no `crm.html`, e clique em
Reclassificar tudo.

---

## Automações de visita marcada

Três filas na aba **Automações**, com o texto pronto e link direto pro WhatsApp:

| Fila | Quando | Para quê |
|---|---|---|
| Confirmar quem vem amanhã | Na véspera | É o que separa visita marcada de visita realizada |
| Avisar quem vem hoje | De manhã | Sai com o endereço junto |
| Quem não apareceu ontem | No dia seguinte | Reabre sem cobrança e já oferece outro dia |

**Estas filas não consomem o teto de 3 repescagens, de propósito.** Repescagem é a loja
correndo atrás de quem sumiu — por isso tem teto. Lembrete é confirmar um compromisso que
o próprio cliente marcou. Travar os dois junto faria a loja deixar de confirmar uma visita
por causa de mensagem antiga, que é o contrário do que se quer.

A trava aqui é outra: **cada lembrete sai uma vez só por cliente**. Quem está marcado como
`nao_cliente` fica de fora.

---

## Repescagem

As filas seguem o mesmo playbook do agente de atendimento, e o painel não deixa furar:

- teto de **3 repescagens por cliente**, no total;
- **uma por dia**, no máximo;
- quem **pediu desconto**, **já comprou**, **não é cliente**, **disse que não tem
  interesse** ou **mandou áudio sem retorno da loja** sai de todas as filas
  automaticamente;
- quem pediu para adiar só volta a aparecer depois de 7 dias.

Um lead entra em **uma** fila só — a mais específica — para ninguém ser chamado duas
vezes no mesmo dia.

| Fila | Quem entra |
|---|---|
| Recebeu a localização e não apareceu | Levou o endereço, não tem pós-venda, sumiu |
| Combinou dia e não voltou | A data marcada passou e não apareceu |
| Perguntou o preço e recebeu áudio | O vazamento nº 1 da operação |
| Parou na numeração | Já escolheu modelo e cor, travou no número |
| Sumiu depois das fotos | Recebeu fotos ou catálogo e não respondeu |

Cada linha já vem com o texto sugerido trazendo **informação nova** na ordem de
prioridade do playbook: preço por escrito → numeração separada em nome dele → entrega
grátis (só DDD 62) → horário da loja. E sempre citando o que o cliente já tinha
escolhido.

O botão **Copiar fila para o Claude** gera a lista pronta para o Claude executar a
repescagem no WhatsApp Web.

> **Manutenção:** a tabela de preços usada para montar a sugestão está em `crm.html`, na
> constante `PRECOS`. Se a loja reajustar, atualize lá — senão a mensagem sai com o valor
> velho. Quando o modelo não estiver na tabela, o texto pede a confirmação do valor em
> vez de inventar.

---

## Limitações conhecidas

- **A ingestão não é automática.** Alguém precisa rodar a varredura do Claude. O CRM não
  fica escutando o WhatsApp sozinho — isso exigiria a API oficial da Meta, com migração
  dos quatro números e custo por conversa.
- **Mensagens recebidas** vêm de `crm_msgs`, que só existe para os dias em que houve
  varredura. Sem varredura no mês, o painel cai para os contadores dos próprios leads
  (que só cobrem quem entrou no mês) e avisa isso embaixo do número.
- **Quem já era cliente antes da primeira varredura** só entra no CRM quando aparecer
  numa varredura ou for cadastrado à mão.
