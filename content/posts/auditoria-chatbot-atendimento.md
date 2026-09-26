---
title: "Auditoria informal a um chatbot de atendimento: rate limiting, CORS e IDOR conceptual"
date: 2026-09-26T10:00:00+01:00
draft: false
tags: ["pentest", "api-security", "web"]
---

## Contexto

Um amigo geria as redes sociais de uma pequena empresa que tinha lançado um
chatbot de atendimento no site, construído sobre um backend ASP.NET com um
modelo de linguagem por trás (Azure AI / Dify, visível no próprio código do
widget). Por curiosidade e para praticar, pedi-lhe autorização informal para
testar a segurança básica do chatbot e da API que o serve, fora de qualquer
contexto profissional ou contratual. Os achados abaixo foram depois reportados
informalmente à empresa através dele.

## Metodologia

1. Reconhecimento do HTML e JavaScript servidos ao browser, à procura de chaves de
   API ou segredos hardcoded no cliente.
2. Inspeção do tráfego de rede via DevTools durante uma conversa normal, para mapear
   o endpoint real que serve as respostas do assistente.
3. Testes de prompt injection e jailbreak contra o próprio modelo.
4. Testes de abuso da API descoberta, fora do browser, via `curl.exe` e PowerShell.

O endpoint identificado:
POST /api/Chat/stream
Content-Type: application/json
Body: { message, user_id, conversation_id, language }


## Achados

### 1. Ausência de rate limiting — Alta

O endpoint aceita pedidos repetidos e consecutivos sem qualquer atraso, bloqueio, ou
resposta 429.

**Evidência.** 20 pedidos consecutivos, disparados em sequência rápida:

```powershell
1..20 | ForEach-Object {
    curl.exe -s -o NUL -w "%{http_code}`n" -X POST "https://<endpoint>/api/Chat/stream" `
        -H "Content-Type: application/json" -H "Accept: text/event-stream" `
        --max-time 5 --data "@payload.json"
}
```

Resultado: 20 respostas `200 OK`, zero bloqueios.

**Impacto.** Cada mensagem processada dispara provavelmente uma chamada paga a um
modelo de linguagem. Sem limite de pedidos, um script simples gera milhares de
chamadas em minutos, esgotando o orçamento de API do cliente ou tornando o serviço
indisponível (negação de serviço por exaustão de recursos).

**Recomendação.** Limite de pedidos por IP e por `conversation_id` no backend
(ex: 10-20 mensagens/minuto por sessão), com resposta 429 ao exceder. Como o site já
usava Cloudflare, isto era configurável diretamente no painel, sem alterar código.

### 2. CORS sem restrição de origem — Alta

O endpoint devolve `Access-Control-Allow-Origin: *` em todas as respostas, incluindo
pedidos com uma origem completamente arbitrária.

**Evidência.**

```powershell
curl.exe -s -I -X OPTIONS "https://<endpoint>/api/Chat/stream" `
    -H "Origin: https://sitequalquer.com" `
    -H "Access-Control-Request-Method: POST"
```

Resposta: `204`, com `Access-Control-Allow-Origin: *` e
`Access-Control-Allow-Methods: POST` confirmados para uma origem não relacionada com
o site do cliente.

**Impacto.** Combinado com o achado 1, qualquer site terceiro pode embutir um script
que chama este endpoint a partir do browser de cada um dos seus visitantes. Isto
transforma os visitantes de um site malicioso ou comprometido num exército
distribuído de pedidos contra a API do cliente, sem que este consiga bloquear por IP
de forma eficaz (os IPs seriam dos visitantes desse site terceiro, não de um atacante
único). O header `Access-Control-Allow-Credentials` não estava presente, o que evita
o cenário mais grave de roubo de sessão via CORS, mas o problema de abuso de custos
mantém-se.

**Recomendação.** Restringir `Access-Control-Allow-Origin` aos domínios legítimos que
embutem o widget, em vez do wildcard.

### 3. `conversation_id` sem validação de posse — Média

O backend aceita qualquer `conversation_id` enviado pelo cliente sem confirmar que
pertence à sessão ou IP que faz o pedido.

**Evidência.** Foi possível continuar uma conversa através de um `conversation_id`
capturado por inspeção de rede, chamando a API diretamente, sem qualquer cookie de
sessão ou token associado.

**Impacto.** Não é explorável por tentativa aleatória (UUID com ~122 bits de
entropia). O risco real é indireto: se um `conversation_id` for exposto por acidente
(log de erro, URL partilhado, link de suporte), qualquer pessoa com esse valor pode
continuar ou influenciar essa conversa específica. Conceptualmente é um IDOR, mesmo
sem vetor prático de exploração imediato.

**Recomendação.** Associar o `conversation_id` a uma sessão validada do lado do
servidor (cookie httpOnly ou token assinado), em vez de confiar só no valor enviado
pelo cliente.

### 4. Mensagens de erro verbosas — Baixa

Pedidos malformados devolvem erros detalhados do ASP.NET, incluindo caminho exato do
campo, número de linha e posição de byte onde a validação falhou.

**Impacto.** Ajuda um atacante a mapear a estrutura interna da API mais depressa, mas
não expõe dados sensíveis por si só.

**Recomendação.** Mensagens de erro genéricas em produção, detalhe completo só nos
logs internos.

## O que resistiu

- Extração do prompt de sistema por pedido direto: recusado corretamente.
- Jailbreak via roleplay ("modo debug"): recusado corretamente.
- Nenhuma chave de API ou segredo encontrado hardcoded no HTML/JS servido ao cliente.

Vale a pena registar isto, um relatório que só lista falhas dá a imagem errada de que
tudo estava mal. O modelo em si, na camada de prompt, estava bem defendido, o
problema estava todo na camada de infraestrutura à volta dele.

## Lições

O erro mais comum aqui não foi de código de aplicação, foi de configuração de
infraestrutura (CORS e rate limiting são normalmente responsabilidade de quem
configura o proxy/CDN, não de quem escreve a lógica do chatbot). Isto reforçou-me que,
ao avaliar a segurança de um produto com IA integrada, a superfície de ataque mais
provável não é o modelo em si, é a API que o expõe. Da próxima vez, começo os testes
de abuso de API mais cedo no processo, em vez de deixar isso para o fim depois dos
testes de prompt injection.
