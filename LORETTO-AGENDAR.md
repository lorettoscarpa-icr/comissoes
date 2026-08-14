# Agendar a Loretto na máquina da loja

Este é o último elo: fazer as rotinas acontecerem sem ninguém lembrar.

---

## O que automatizar, e o que não

Antes de qualquer script, a decisão que importa.

### Automatizar: a leitura

A varredura só **lê** as conversas. Nenhum risco, e é ela que resolve o problema real — o
painel ficar velho porque ninguém rodou. Automatize sem medo.

### Não automatizar: o envio

Duas razões, e a segunda é a que pesa.

**1. O WhatsApp bane número por automação.** Envio em lote sem gente na frente é o padrão
que a Meta procura para banir. Esses quatro números **são o negócio** — perder um custa
mais do que todo o tempo que a automação economizaria em um ano.

**2. O playbook de vocês já proíbe.** A regra é conferir o destinatário *imediatamente
antes de cada envio*, porque a vendedora pode estar usando a mesma janela e trocar de
conversa no meio da operação — e isso já quase mandou fotos para o contato errado. Um
script sem ninguém olhando não faz esse julgamento.

### O que sobra é quase tudo

O trabalho difícil já é automático: **decidir**. Quem chamar, em que ordem, com que
informação, respeitando teto e bloqueios, na voz de cada vendedora. Isso o CRM faz sozinho,
24 horas, sem Claude nenhum.

O que fica humano é **tocar em enviar** — cinco minutos, com a lista pronta na tela.

> Não é meia automação. É a automação inteira, com a última milha protegida de propósito.

---

## O que a máquina da loja faz

| Hora | O quê |
|---|---|
| **8h30** | Varredura das quatro contas, uma por vez. Alimenta o CRM. |
| **9h00** | Abre o painel na aba *Loretto* e avisa que os lotes estão prontos. |
| **10h00** | Segundo aviso, se ainda não rodaram. |

O envio acontece quando alguém abre e toca. O painel mostra **"Ainda não rodou hoje"** em
vermelho enquanto isso não acontece, e **"Rodou hoje às 10h02 · 39 clientes"** depois.

---

## Instalar

### Requisitos

- Um computador que fica ligado no horário comercial (o do caixa serve)
- Chrome com a extensão **Claude in Chrome**
- As quatro contas de WhatsApp Web logadas — **em perfis separados do Chrome**, senão elas
  se atropelam

Crie um perfil por vendedora, uma vez só:

```
chrome.exe --profile-directory="Jhennifer"
chrome.exe --profile-directory="Ialey"
chrome.exe --profile-directory="Michelly"
chrome.exe --profile-directory="Natalia"
```

Em cada um, abra `web.whatsapp.com` e leia o QR do celular daquela vendedora. Feito isso, a
sessão fica salva no perfil.

### Windows — Agendador de Tarefas

Salve como `loretto-manha.ps1`:

```powershell
# Loretto — abre o painel e as contas para a varredura da manhã
$crm = "https://lorettoscarpa-icr.github.io/crm/"

# 1) abre o CRM na aba de rotinas
Start-Process chrome.exe $crm

# 2) abre uma janela por vendedora, com 40s entre elas.
#    Uma de cada vez: várias janelas com a extensão na mesma conta derrubam a extensão.
foreach ($v in @("Jhennifer","Ialey","Michelly","Natalia")) {
  Start-Process chrome.exe "--profile-directory=`"$v`" https://web.whatsapp.com"
  Start-Sleep -Seconds 40
}

# 3) avisa quem estiver na loja
[void][System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms")
[System.Windows.Forms.MessageBox]::Show(
  "Loretto: os lotes do dia estao prontos no painel.",
  "Loretto - Loretto Scarpa")
```

Agendar:

```powershell
schtasks /create /tn "Loretto manha" /tr "powershell -File C:\loretto\loretto-manha.ps1" /sc daily /st 08:30
```

### Mac — cron

```bash
crontab -e
```

```cron
30 8 * * 1-6  /Users/loja/loretto/manha.sh
```

`manha.sh`:

```bash
#!/bin/bash
open "https://lorettoscarpa-icr.github.io/crm/"
for v in Jhennifer Ialey Michelly Natalia; do
  open -na "Google Chrome" --args --profile-directory="$v" "https://web.whatsapp.com"
  sleep 40
done
osascript -e 'display notification "Os lotes do dia estão prontos" with title "Loretto"'
```

---

## O turno da Loretto

Com as janelas abertas, quem estiver na máquina abre a extensão e diz:

> Você é a Loretto. Leia o `LORETTO.md` e o `CRM-WHATSAPP.md` do repositório `crm`.
> Faça a varredura da conta da **Jhennifer** e importe no CRM. Depois me mostre o lote
> de hoje.

Ela varre, importa, e apresenta os lotes. A pessoa revisa e envia.

No fim, registrar — é o que faz o painel parar de reclamar:

```js
await CRM.registrarRotina('manha', 39)
```

---

## Como saber se está funcionando

O painel responde sozinho, na aba **Loretto**:

- **"Rodou hoje às 10h02 · 39 clientes"** — em ordem
- **"Ainda não rodou hoje"** em vermelho — a máquina não acordou, o Chrome não abriu, ou
  ninguém tocou
- **"Os dados podem estar velhos"** no topo, com o nome das contas sem varredura recente

`CRM.pendentesDeHoje()` devolve o que falta agora.

Sem esse registro, "rotina automática" seria promessa: ninguém saberia se a máquina acordou,
se o Chrome abriu, se a conta caiu. Com ele, o silêncio vira aviso.

---

## Se um dia não rodar

Nada quebra e nada se perde. O lote do dia seguinte vem maior, na mesma ordem de
prioridade — quem espera resposta continua em primeiro, quem esfriou há mais tempo desce.

O único custo é o tempo: quem esfriou ontem responde melhor do que quem esfriou semana
passada. Por isso o painel reclama.
