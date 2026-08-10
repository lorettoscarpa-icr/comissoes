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
| `posVendaEnviado` | **Texto exato** — procurar a mensagem de pós-venda na conversa |
| `canalVenda` | IA — fechou na loja ou pediu entrega |

---

## Quem entra e o que cada uma vê

Cada vendedora tem o **próprio login**. O CRM descobre quem é em três degraus:

1. o que estiver escrito em `crm_config/usuarios`;
2. o e-mail parecer com o nome de uma vendedora (`jhennifer.cunha@…`);
3. sem nada disso, entra como **gestão** e a tela avisa que falta configurar — assim a
   conta que já usava o painel não trava de um dia para o outro.

Para cadastrar, no console da aba do CRM (logada como gestão):

```js
await CRM.definirUsuario('michelly@loretto.com', 'Michelly')
await CRM.definirUsuario('larissa@loretto.com', '', 'gestao')
```

| Papel | Vê |
|---|---|
| **Vendedora** | Abre em *Meu dia*. Só a carteira dela, em todas as telas. Os comparativos mostram só a linha dela. Sem Cruzamento (faturamento da loja) e sem Importar |
| **Gestão** | Abre no *Painel*. Tudo, incluindo o comparativo das quatro |

O recorte é feito num único ponto — a função que lista os leads — para não sobrar tela
sem filtro. Trocar o padrão (deixar a vendedora ver o comparativo das quatro) é decisão de
gestão, não de código.

> **Isto é conforto de uso, não segurança.** Filtro no navegador não barra quem abrir o
> console. Quem barra de verdade são as regras do Firestore:
>
> ```
> match /crm_leads/{tel} {
>   allow read, write: if request.auth != null;   // troque por uma checagem de carteira
> }                                                // quando cada vendedora tiver login
> ```

---

## Confiança nos dados

Duas coisas que o CRM faz para não mentir em silêncio.

**A dona do lead nunca é trocada pela varredura.** O telefone é a chave única, então a
leitura de outra conta sobrescrevia o registro e o cliente mudava de carteira sozinho,
levando comissão junto — e, com o recorte por vendedora, sumindo do *Meu dia* de uma e
aparecendo no da outra. Agora divergência abre um **conflito** no painel de gestão, quem
chegou primeiro continua dono, e a decisão humana trava a varredura seguinte.

**O painel diz quando os dados foram atualizados, por conta.** Sem isso, cinco dias sem
varredura davam uma tela que parecia normal. O carimbo é por vendedora porque uma conta
varrida ontem e outra da semana passada produzem um painel coerente e falso. Passando de
2 dias, aparece o aviso.

---

## Catálogo, preço e estoque

O preço saiu do código para `crm_config/catalogo`:

```js
await CRM.definirCatalogo([
  {modelo:'Loafer Dubai',  preco:349.90, estoque:true},
  {modelo:'Slipper Ibiza', preco:289.90, estoque:false}
])
```

Enquanto ninguém cadastrar, vale a tabela embutida no `crm.html` — mas ela tem uma
ambiguidade real que o catálogo resolve: **um modelo cujo nome casa com duas regras recebe
o preço da primeira**. "Bota Verona" sai por R$ 299,90 (regra dos sportfinos) e não por
R$ 329,90 (regra das botas), porque `verona` é testado antes de `bota`. Com o catálogo
cadastrado, o nome exato vence e a dúvida acaba.

`estoque: false` marca produto que **não pode ser oferecido**, como o Slipper Ibiza.

---

## Data de visita e data de compra são coisas diferentes

"Vou passar aí sábado" e "compro dia 20" são compromissos distintos, e o campo
`tipoAgendamento` separa os dois:

| Tipo | O que muda |
|---|---|
| **Vir à loja** | Mensagem leva o endereço e o horário. Falta vira cobrança. |
| **Fechar pedido** | Mensagem fala em finalizar o pedido, **sem endereço**. Não gera cobrança de falta, porque ele nunca prometeu aparecer. |

Sem essa separação, quem marcou compra para receber em casa recebia o endereço da loja e
era contado como no-show.

---

## Ordem das filas: o mais recente primeiro

O briefing pede "ordem sempre do mais recente para o mais antigo", e a direção **muda
conforme a ação**:

- **Está esperando resposta** → quem espera **há mais tempo** primeiro. É dívida acumulando.
- **Filas frias** (sumiu, não apareceu, parou na numeração) → o **mais recente** primeiro.
  Quanto mais fresco o esfriamento, maior a chance de resposta.

> **Corrigido:** tudo vinha do mais antigo para o mais novo, então a repescagem começava
> por quem sumiu há meses e deixava para depois quem sumiu ontem.

### Quem espera resposta não é conversa fria

O filtro de cada fila olhava a última mensagem **da loja**, então alguém que respondeu
ontem continuava parecendo frio porque a loja tinha escrito há uma semana — e receberia
*"ainda tem interesse?"* logo depois de fazer uma pergunta. Na base de teste isso pegava
**38 de 87** da fila.

Agora quem escreveu por último sai das filas frias e vai para a ação de **responder**, que
tem prioridade sobre tudo.

### Quando um lead entra na fila fria

Depois de **2 dias** de silêncio (`DIAS_ESFRIOU`). Quem esfriou ontem só entra amanhã — de
propósito, para não cobrar quem ainda está pensando. A ação de *responder* não tem
carência: aparece na hora.

---

## Meu dia — o CRM diz o que fazer

A vendedora não escolhe fila nem aba. Abre em **Meu dia**, vê uma linha por cliente na
ordem em que vale a pena atacar, e desce de cima para baixo.

Cada linha traz **o que fazer**, **por quê** (a evidência) e, quando dá para saber, **o
texto pronto** — com o link que abre o WhatsApp já com a mensagem digitada.

### A ordem, e por que ela é essa

| # | Ação | Quando | Consome o teto? |
|---|---|---|---|
| 1 | **Está esperando resposta** | O cliente escreveu por último e a loja não respondeu | Não |
| 2 | Vem à loja hoje | Visita marcada para hoje | Não |
| 3 | Perguntou o preço e recebeu áudio | Situação `preco_so_audio` | Sim |
| 4 | Vem à loja amanhã | Visita marcada para amanhã | Não |
| 5 | Marcou e não apareceu | A data passou sem comparecimento | Não |
| 6 | Recebeu o endereço e não veio | Levou a localização e sumiu | Sim |
| 7 | Já escolheu tudo, falta marcar o dia | Modelo, cor e numeração fechados, sem data | Sim |
| 8 | Parou na numeração | Travou na última pergunta antes de fechar | Sim |
| 9 | Sumiu depois das fotos | Recebeu catálogo e não respondeu | Sim |
| 10 | Comprou faz tempo | Venda há 60 dias ou mais | Não |

Duas decisões deram essa ordem:

**Responder quem espera vem antes de qualquer repescagem.** O diagnóstico mediu cliente
perguntando às 10h54 e sendo respondido às 14h25. Correr atrás de quem sumiu enquanto
alguém espera na fila é trocar dinheiro certo por dinheiro incerto.

**Preço que ficou só em áudio vem logo depois.** Foram 122 casos numa única varredura — o
vazamento mais caro e o mais barato de fechar.

### Regras que a lista respeita

- **Uma ação por cliente, nunca duas.** Senão a mesma pessoa recebe duas mensagens no
  mesmo dia — exatamente o que o teto existe para impedir.
- **Só o que é abordagem fria consome o teto de 3.** Responder alguém que escreveu, ou
  confirmar uma visita que o cliente marcou, não é repescagem.
- Quem está marcado como `não é cliente` nunca aparece.
- A ação de *Está esperando resposta* **não sugere texto** — não dá para saber o que ele
  perguntou. Ela abre a conversa.
- A lista mostra as 30 mais urgentes e **diz quantas ficaram de fora** — nada é truncado em
  silêncio.

---

## Repescagem sem mandar duas vezes

Quando a vendedora clica em **Abrir no WhatsApp**, o CRM grava a repescagem **na hora**,
com estado `aberto` — sem esperar que ela volte para confirmar.

O motivo é prático: ela abre o WhatsApp, manda, atende o próximo cliente e não volta. Um
sistema que depende desse retorno acha que a mensagem não saiu e oferece o mesmo cliente
de novo amanhã — **o cliente recebe duas vezes**, quebrando em silêncio a regra de uma por
dia que o CRM existe para garantir.

| Estado | Conta no teto de 3? |
|---|---|
| `aberto` — clicou e abriu o WhatsApp | **Sim** |
| `confirmado` — a varredura viu a mensagem na conversa | Sim |
| `nao_enviado` — abriu e não mandou | Não, volta para a fila |

Contar o `aberto` erra de propósito para o lado de **não incomodar o cliente**, que é o
erro barato. A linha continua na tela, apagada, com o botão **Não enviei** — para o caso de
ela abrir e desistir.

A varredura reconcilia depois, devolvendo o veredito por data:

```json
{"repescagens": [{"data": "2026-08-09", "estado": "confirmado"}]}
```

Os lembretes de visita funcionam igual, mesclados por tipo e data.

---

## Classificação automática por departamento

O departamento do lead é lido da **primeira mensagem do cliente** (e do **nome do
contato**), por regra de texto — não por alguém marcar nada, e sem IA.

Cada campanha entra com um texto próprio e fixo. Casar padrão nisso é exato, custa zero e
nunca inventa. IA num problema desses só adicionaria erro.

### Os departamentos

| Departamento | Grupo | Texto de entrada |
|---|---|---|
| **Público Frio** | Anúncios | "sou de / estou em Goiânia" + interesse |
| **Remarketing** | Anúncios | "vim **do** Instagram e quero saber mais informações sobre…" |
| **Anúncio s/ campanha** | Anúncios | Tem cara de anúncio, sem marcador de qual |
| **Bio do Instagram** | Instagram | "vim pelo link da bio e quero receber o catálogo" |
| **Direct do Instagram** | Instagram | "gostei de um sapato que vi **no** Instagram" |
| **Cupom de Desconto** | Instagram | "tenho 5% de desconto para realizar a minha primeira compra" |
| **Clientes** | Orgânico | O **nome do contato** tem "Cliente" — ex.: "Gregory Cliente" |
| **Indicação** | Orgânico | "me indicou", "meu amigo comprou aí" |
| **Lead Orgânico** | Orgânico | Não veio por nenhum departamento |
| **Não identificada** | — | Sem a primeira mensagem, não dá para dizer |

### Os quatro canais

O painel abre com a leitura que a operação acompanha — leads e conversão de cada canal:

| Canal | O que entra |
|---|---|
| **Anúncios** | Público Frio · Remarketing |
| **Instagram** | Bio · Direct · Cupom de Desconto |
| **Clientes** | Já compraram, número salvo |
| **Lead Orgânico** | Não veio por nenhum departamento |

Repare que **Clientes é um departamento dentro do grupo Orgânico, mas aparece como canal
próprio**: recompra e lead novo não se administram do mesmo jeito, e misturar os dois
esconde os dois.

O **grupo** existe por um motivo prático: nove cores num gráfico não se distinguem. A cor
carrega o grupo e o nome do departamento vem escrito ao lado — identidade nunca fica só na
cor.

### Três decisões que valem saber

**O nome do contato vence a mensagem.** "Gregory Cliente" escrevendo por um anúncio de
remarketing é classificado como **Cliente**, não como Remarketing. Saber que é recompra
muda o atendimento inteiro, e vale mais do que por qual anúncio a pessoa voltou. A regra
exige a palavra inteira — "Clientelismo Souza" não casa.

**Anúncio sem campanha não vira Público Frio.** Quando o texto tem cara de anúncio mas
falta o marcador que diz de qual campanha veio, o lead fica explicitamente em *Anúncio s/
campanha* e cai na fila de conferência. Chutar aqui contaminaria a leitura de verba, que é
justamente o que esses números existem para responder.

**A ordem das regras importa, e aqui ela evita um erro caro.** A mensagem do Direct
termina em *"quero saber mais informações"* — exatamente o gatilho da regra genérica de
anúncio. Se o Direct não for testado antes, **todo lead de Direct é contado como anúncio
pago**, e a leitura de verba passa a somar lead que não custou nada. O que separa o Direct
do Remarketing é uma preposição: "vi **no** Instagram" contra "vim **do** Instagram".

Cupom vem antes de Bio, porque as duas terminam pedindo o catálogo. A primeira regra que
casar vence.

O texto do Público Frio **já teve variações** ("sou de" / "estou em", "tenho interesse" /
"quero saber"), então a regra cobre as duas formas em vez de casar uma frase literal — se
mudar de novo, ela continua pegando.

### Garantias

- **Departamento definido na mão nunca é sobrescrito.** Mexeu na ficha, trava.
- **O que nenhuma regra classificou cai na fila de conferência** (aba Automações), junto
  com o que casou só com confiança média. A automação não erra em silêncio.
- **`Reclassificar tudo`** roda as regras sobre a base inteira, então padrão novo vale
  também para o histórico — não só daqui pra frente.

Para adicionar um padrão: inclua a frase em `REGRAS_ORIGEM`, no `crm.html`, e clique em
Reclassificar tudo.

---

## Como a venda é medida

### O marcador de compra — e a dedução do comparecimento

A mensagem de pós-venda é **automática**, disparada pelo sistema em toda compra. Isso a
torna um marcador confiável nos dois sentidos: quem tem, comprou; **quem não tem, não
comprou**. Por isso ninguém precisa digitar "veio / não veio".

Trechos procurados na conversa (o nome da vendedora e o do cliente variam, estes não):
`Muito obrigada pela sua compra` · `Comunidade VIP` · `Conte comigo sempre que você
precisar` · `novidades em primeira mão`.

Duas armadilhas tratadas, porque uma dedução errada aqui inventa no-show que não houve:

**Ausência só vale se alguém leu.** Se a conta daquela vendedora não foi varrida depois
da visita, o CRM responde *"ainda não dá para saber"* em vez de *"não veio"*. Uma semana
sem varredura marcaria a loja inteira como falta.

**Pós-venda prova compra, não presença.** Venda por entrega também recebe a mensagem. A
dedução de "veio" exige que a rota não seja entrega.

Há **um dia de folga** depois da data combinada, porque o pós-venda pode sair só no fim do
expediente. Varredura no mesmo dia da visita ainda não conclui; no dia seguinte, conclui.

| O que o CRM vê | O que ele conclui |
|---|---|
| Pós-venda na conversa, rota não é entrega | **Veio e comprou** |
| Sem pós-venda, conta varrida depois da data | **Não veio** |
| Sem pós-venda, conta não varrida | *Ainda não dá para saber* |
| Alguém marcou na mão | Vale a marcação, sempre |

Os botões na Agenda continuam lá, mas só para **corrigir exceção** — quem veio,
experimentou e não levou nada, por exemplo.

### Detalhes do marcador

A **mensagem automática de pós-venda** é marcador determinístico de compra fechada — o
mesmo raciocínio do template de anúncio. Quem tem esse texto na conversa comprou; quem
recebeu a localização e **não** tem é exatamente quem vale chamar de volta.

O Claude procura por `Muito obrigada pela sua compra` ou `Comunidade VIP` durante a
varredura e manda `posVendaEnviado: true`. O CRM move o lead para *vendido* sozinho.

### As duas rotas de venda

| Rota | O que é |
|---|---|
| **Na loja** | O cliente veio, experimentou e fechou |
| **Por entrega** | Fechou sem pisar na loja, recebeu em casa |

Isso não é detalhe de cadastro. **Venda por entrega não é comparecimento.** Contar as duas
juntas esconde qual rota cada vendedora domina — e uma vendedora que fatura bem só por
entrega tem um problema de convencimento presencial que fica invisível.

> **Correção feita nesta versão:** o funil deduzia o comparecimento pela posição no funil —
> quem estava em *vendido* contava como tendo passado por *compareceu*. Toda venda de
> entrega era contada como visita à loja e a taxa de comparecimento ficava inflada. Agora
> **Compareceu conta só marcação explícita**.

Como o caminho da entrega pula a loja, o funil pode **crescer** em *Vendido* em relação a
*Compareceu*. Não é defeito: a barra vem marcada com `+N por entrega`.

### Conversão em três níveis

| Nível | O que diagnostica |
|---|---|
| **Lead → visita** | Convencimento no WhatsApp |
| **Visita → venda** | A loja: estoque, numeração, atendimento presencial |
| **Lead → venda** | O resultado final |

São problemas com soluções opostas. Quem tem lead→visita baixa precisa melhorar a conversa;
quem tem visita→venda baixa precisa olhar estoque e atendimento na loja.

---

## Cruzamento com os outros sistemas

Aba **Cruzamento**. Os quatro sistemas da loja usam **o mesmo Firebase**
(`loretto-scarpa-l-icr`) e **os mesmos nomes de vendedora**, então não há integração a
fazer — é só ler. **Somente leitura: o CRM nunca escreve nas coleções dos outros.**

| Sistema | Repositório | Coleções lidas |
|---|---|---|
| Central de Pagamentos | `comissoes` | `fechamentos`, `metas`, `atingimentos` |
| Central de Entregas | `entregas` | `entregas` |
| Central de Metas | `metas` | `ls_metas`, `ls_historico` |

### A junção exata: entregas ↔ leads pelo telefone

A Central de Entregas grava o telefone do cliente em `contato`. É a **mesma chave** do CRM —
e a única ligação exata entre as bases; todo o resto é comparação de totais.

O telefone é digitado à mão lá, então os formatos variam. O CRM canoniza os dois lados para
`DDD + 9 + 8 dígitos` antes de comparar — inclusive reinserindo o nono dígito em registros
no formato antigo. Verificado com cinco formatos diferentes do mesmo número
(`5562…`, `62…`, `(62) 9…`, `+55 62 9…` e sem o nono dígito): todos casam.

Disso saem dois números que nenhuma fonte tem sozinha:

- **Entregas com lead no CRM** — a venda nasceu de uma conversa rastreada.
- **Entregas sem lead nenhum** — ou a varredura não capturou, ou a venda não veio do
  WhatsApp. É a medida direta do buraco de captura.

O botão **Marcar como venda por entrega** preenche rota, valor e etapa nos leads casados.
Só age sobre entrega com status `entregue` — pedido em rota ainda pode ser cancelado, e
marcar venda antes da hora infla o número. A Central de Entregas não é alterada.

### As três contagens da mesma venda

| Fonte | O que conta |
|---|---|
| **Kigi** (`fechamentos`) | O que passou no sistema de vendas |
| **Anotado** (`ls_historico`) | O que a vendedora contou, por dia e categoria |
| **CRM** | O que veio de conversa no WhatsApp |

Divergência grande entre as duas primeiras é **lançamento faltando** — de um dos lados. A
tabela marca a diferença em vermelho acima de 3 peças.

### Por que os dois números não batem (e não deveriam)

| Fonte | Mede |
|---|---|
| **Central** (vem do Kigi) | Tudo que a loja vendeu, inclusive quem entrou pela porta |
| **CRM** (vem do WhatsApp) | Só o que a varredura conseguiu atribuir a uma conversa |

A diferença **é a informação**: é a fatia do faturamento que não passou pelo WhatsApp,
somada ao que a varredura deixou escapar.

> **Cuidado com a unidade.** A Central conta **peças** e **valor**; o `totalVendas` de lá é
> número de linhas do arquivo, não de clientes. O CRM conta **clientes**. Por isso o
> cruzamento é feito em valor e em peças — nunca em "quantidade de vendas", que significa
> coisas diferentes nos dois lados.

### O que a aba mostra

- **Atribuído ao WhatsApp** — quanto do faturamento real o CRM conseguiu ligar a uma
  conversa. É um **piso**, não o número exato: se a varredura não rodou, ele desaba sem que
  a loja tenha vendido menos pelo WhatsApp.
- **Mensagens recebidas por peça vendida**, por vendedora — o custo em conversa de cada
  venda. Número alto é atendimento caro: muita conversa para pouca peça.
- **Dia a dia** — mensagens recebidas × peças vendidas. Dia com muita mensagem e pouca peça
  é gargalo. Usa o `_porDia` que a Central já grava por vendedora e categoria.
- **Metas do mês** — meta, realizado e quanto falta, por categoria, ao lado de quantos leads
  já receberam preço e ainda não fecharam. É o estoque de conversa que ainda pode virar meta
  batida.

### Se a aba aparecer vazia

Ou não existe fechamento da Central no mês (é só subir o Excel do Kigi lá), ou as regras do
Firestore não liberam leitura de `fechamentos`, `metas` e `atingimentos` para este usuário —
a aba diz qual dos dois é.

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
