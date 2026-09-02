# ADR-004: HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito. Reunião técnica, `[09:19]` a `[09:23]`.

## Contexto

O webhook envia dados de pedidos para um endpoint **fora da nossa infraestrutura**. O cliente
precisa conseguir provar duas coisas ao receber a requisição:

1. que ela veio realmente de nós;
2. que ninguém adulterou o payload no caminho.

## Decisão

**HMAC-SHA256** sobre o corpo do request, enviado no header **`X-Signature`**. O cliente recalcula
do lado dele e compara.

> *"HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso."*
> - Sofia, `[09:20]`

**Cada endpoint tem a sua própria secret.** Não existe secret global da plataforma.

> *"Se vaza uma, vaza tudo."* - Sofia, `[09:21]`

A secret é **gerada por nós** e devolvida ao cliente **apenas na criação** do webhook.

**Rotação suportada**: o cliente pede uma nova secret pela API e a **anterior continua válida por
24 horas**, para ele migrar os sistemas dele sem downtime. Passado o prazo, a antiga morre.

Complementos de segurança decididos junto:

- **HTTPS obrigatório** na URL cadastrada; `http` é recusado na validação de schema
- **Teto de 64KB** por payload; acima disso o evento **falha**, não é truncado

## Alternativas consideradas

### Secret única global da plataforma

Descartada. Um vazamento comprometeria a integração de **todos** os clientes de uma vez. O risco é
concreto: já houve cliente que vazou secret em log de aplicação (`[09:22]` Diego).

### Rotação sem grace period

Descartada implicitamente. Trocar a secret de forma instantânea quebraria a integração do cliente
no intervalo entre a rotação e o deploy dele. As 24 horas existem para cobrir esse intervalo.

### mTLS

Não discutida na reunião. Registrada aqui como alternativa plausível: daria autenticação mútua
mais forte, mas exige gestão de certificados dos dois lados, o que elevaria muito a barreira de
integração para clientes B2B que hoje só sabem receber um POST.

## Consequências

### Positivas

- Padrão de mercado, com biblioteca pronta em qualquer linguagem
- Vazamento fica **contido a um cliente**
- Rotação sem downtime para o cliente
- HTTPS e limite de tamanho fecham dois vetores simples de problema

### Negativas

- Precisamos **armazenar e gerenciar N secrets**, uma por endpoint
- Durante a janela de 24h, **duas secrets** são válidas ao mesmo tempo, o que exige cuidado na
  verificação e no armazenamento
- Geração de secret e HMAC são código sensível: a reunião reservou **2 dias úteis** de revisão de
  segurança antes do deploy (`[09:46]` Sofia)
