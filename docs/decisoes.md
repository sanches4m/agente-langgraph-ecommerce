# Decisões de arquitetura (ADRs)

Cada decisão importante, com o contexto, o que foi escolhido, as alternativas e o preço da
escolha. Decisão nova entra no fim, com o próximo número. Decisão que mudou não é apagada: fica
marcada como **Substituída por ADR-0XX**.

Formato: **Contexto** (o problema), **Decisão**, **Alternativas** (o que foi descartado e por
quê), **Consequências** (o que se ganha e o que se paga).

Visão geral do sistema: [documento_de_arquitetura.md](documento_de_arquitetura.md).

| ADR | Decisão | Situação |
|---|---|---|
| 001 | Orquestrador + especialistas por área | Aceita |
| 002 | A IA só escolhe a ação; o texto é fixo | Aceita |
| 003 | Especialista obrigado a chamar ferramenta | Aceita |
| 004 | Confirmação humana em toda ação, sobre a lista que a pessoa viu | Aceita |
| 005 | UpSeller e impressora atendidas por um PC local com fila no banco | Aceita |
| 006 | Frases prontas antes da IA e aprendizado automático | Aceita |
| 007 | Organizar envio e imprimir etiqueta são pedidos separados | Aceita |
| 008 | Agente não inicia conversa (função desligada) | Aceita |
| 009 | Rastreio no LangSmith desligado por padrão | Aceita |
| 010 | Vercel com atendimento síncrono | Aceita |
| 011 | Postgres (Supabase) como fonte da verdade, inclusive da conversa | Aceita |
| 012 | Modelos em fila com reserva, provedor isolado numa função | Aceita |

---

## ADR-001: Orquestrador + especialistas por área

**Contexto.** São 17 ferramentas em 4 áreas (Shopee/estoque, notas, envios, relatórios). Com
todas na mão de um modelo só, ele confunde ferramentas parecidas (ex.: "pedidos a enviar" em
`consultar_pedidos` x `pedidos_por_situacao`) e a instrução fica enorme.

**Decisão.** Um orquestrador (modelo leve, saída estruturada `Rota`) escolhe a área; cada
especialista só enxerga as ferramentas da sua área e tem instrução própria. Montado em LangGraph.

**Alternativas.** Agente único com todas as ferramentas: mais simples, porém menos preciso e
mais caro por chamada. Classificador sem IA (palavras-chave): não entende frase livre.

**Consequências.** + Precisão e instruções curtas; área nova = especialista novo. + O
orquestrador pode usar um modelo mais barato. − Duas chamadas de IA numa frase livre (mitigado
pelo ADR-006). − Pedido que mistura áreas ("vendas e emitir") atende uma só.

## ADR-002: A IA só escolhe a ação; o texto é fixo

**Contexto.** A resposta vai para quem opera a loja e decide com base nela (quantos pedidos,
quanto vendeu). Um número ou status inventado causa prejuízo.

**Decisão.** A IA escolhe a ferramenta e os argumentos. A ferramenta devolve o texto final, montado
com os modelos de `textos.py`, e o nó `resposta_pronta` manda esse texto sem reescrever.

**Alternativas.** Deixar a IA redigir a resposta a partir do resultado (padrão comum): mais
natural, mas abre espaço para número errado e muda o tom a cada vez.

**Consequências.** + Resposta sempre verdadeira e igual; texto testável. + Menos tokens. −
Respostas menos "conversadas"; frase nova precisa de texto novo no código.

## ADR-003: Especialista obrigado a chamar ferramenta

**Contexto.** Incidente real: pedido de "organizar envio" foi "respondido" pelo especialista
copiando um erro antigo da conversa, com número de pedido inventado, sem executar nada.

**Decisão.** Especialistas usam `tool_choice="any"`: não podem responder em texto. Do histórico,
só o texto das rodadas anteriores vai para o modelo (sem as chamadas de ferramenta antigas).

**Alternativas.** Reforçar a instrução ("sempre use a ferramenta"): não garante. Validar a
resposta depois: detecta tarde.

**Consequências.** + Elimina a classe de erro "fingiu que fez". − Frase que não pede ferramenta
nenhuma precisa ir para `fora_da_loja` já no orquestrador.

## ADR-004: Confirmação humana em toda ação, sobre a lista que a pessoa viu

**Contexto.** Emitir nota, organizar envio, imprimir, mexer em estoque e zerar anúncio têm efeito
real e às vezes irreversível. Entre a pergunta e o "sim" podem chegar pedidos novos.

**Decisão.** A ferramenta pausa o grafo (`interrupt`) com o texto, os botões e os **dados** da
pergunta. O "sim" retoma com esses dados e a ação usa só o que foi mostrado; se a lista mudou,
pergunta de novo. Qualquer frase que não é resposta cancela a pergunta e vira pedido novo.

**Alternativas.** Confirmar só por texto: mais erro de digitação. Agir sem confirmar nas ações
"seguras": a fronteira do que é seguro muda com o tempo.

**Consequências.** + Nenhuma ação sem consentimento explícito e informado. + A conversa fica
salva no banco, então o "sim" pode vir minutos depois. − Um clique a mais em cada ação.

## ADR-005: UpSeller e impressora atendidas por um PC local com fila no banco

**Contexto.** A Shopee não oferece emissão de nota pela API; o emissor (UpSeller) não tem API.
Testado em 20/10/2026: rodando no GitHub Actions, a UpSeller recusa a sessão e pede código por
e-mail a cada IP novo. A impressora térmica é física.

**Decisão.** Um PC Windows da loja roda o vigia (tarefa agendada). O agente põe tarefas na
tabela `tarefas_pc`; o vigia pega a cada minuto, executa (robô Selenium com perfil do Chrome
logado, ou impressão acompanhando a fila do Windows), marca o resultado e chama a API para o
agente conferir e avisar. O vigia grava sinal de vida.

**Alternativas.** Robô na nuvem (Actions, Cloud Run): bloqueado pela UpSeller. Impressão em
nuvem: a térmica não tem. Trocar de emissor: fora do controle agora.

**Consequências.** + Funciona com sistemas sem API e com hardware local. + A fila desacopla:
PC desligado = tarefa espera. − Depende do PC ligado (o agente avisa quando não há sinal). −
O robô quebra se a tela da UpSeller mudar. − Uma loja nova precisa do seu PC local.

## ADR-006: Frases prontas antes da IA e aprendizado automático

**Contexto.** A maioria das mensagens são as mesmas poucas frases ("emitir", "imprimir",
"vendas hoje"). Passar todas pela IA custa tempo, chamadas e abre chance de erro sem motivo.

**Decisão.** Ordem de resolução: texto fixo, admin, resposta a pergunta aberta, atalho fixo
(`textos.ATALHOS`), atalho aprendido, "imprim..." direto em envios, e só então a IA. Uma frase
vira atalho aprendido sozinha quando a IA a entendeu **pelo menos 5 vezes, em 3 dias
diferentes, 100% das vezes com a mesma ação**. Se uma vez foi diferente, ou se a ação tem um
número que não está na frase (ex.: `mes=10` em "lucro do mês"), espera o administrador.

**Alternativas.** Cache de respostas da IA: guardaria resposta velha. Regras só manuais: o
administrador viraria gargalo.

**Consequências.** + Respostas mais rápidas e mais estáveis com o tempo; 100% das mensagens
de 30/10/2026 saíram sem IA. − Um atalho aprendido errado se repete (mitigado pelos 100% e pelo
comando `esquece`).

## ADR-007: Organizar envio e imprimir etiqueta são pedidos separados

**Contexto.** A Shopee só libera a etiqueta alguns minutos depois de o envio ser organizado.
Juntar as duas coisas fazia a impressão falhar ou esperar dentro da função.

**Decisão.** "Organizar envio" só organiza e baixa o estoque, e deixa um aviso pendente. O vigia
pede a conferência a cada volta; quando a Shopee libera, sai o aviso "etiquetas prontas" com o
botão [Imprimir]. Imprimir é outro pedido, com as opções novas / segunda via / tudo.

**Consequências.** + Nenhuma espera dentro da função; reimpressão e segunda via ficam simples.
− Um passo a mais para a pessoa (reduzido pelo botão no aviso).

## ADR-008: Agente não inicia conversa (função desligada)

**Contexto.** Pela Meta, uma empresa pode iniciar conversa com modelo de mensagem aprovado; fora
disso, só responde dentro da janela aberta pela pessoa. Notificações automáticas (ex.: "chegou
pedido") não estão no escopo atual.

**Decisão.** A função de iniciar conversa fica desligada. O agente responde e manda avisos
**só sobre algo que a pessoa pediu** (notas emitidas, etiquetas prontas, impressão feita), com
validade de 2 horas.

**Consequências.** + Nenhuma mensagem fora de hora, sem modelos para aprovar. − Quem quiser saber
de pedido novo pergunta. Ligar depois = modelos aprovados na Meta + um disparador.

## ADR-009: Rastreio no LangSmith desligado por padrão

**Contexto.** O rastreio grava entrada e saída de cada passo, inclusive dados de pedidos e às
vezes nome de comprador. As medidas do dia a dia já ficam na tabela `registros`.

**Decisão.** Desligado; liga-se por variável no Vercel para investigar um caso e desliga depois.
As evals usam o LangSmith sempre (só frases de teste, sem dados de cliente).

**Consequências.** + Privacidade por padrão. − Para investigar um caso passado, só há o que a
tabela `registros` guardou.

## ADR-010: Vercel com atendimento síncrono

**Contexto.** Precisa de uma URL pública com HTTPS para o webhook, deploy a cada merge e quase
nenhuma operação. O volume é de dezenas de mensagens por dia.

**Decisão.** Uma função Python (FastAPI) no Vercel. Como a função congela ao responder, todo o
atendimento acontece antes do 200 à Meta. Esperas longas ficam com o vigia (ADR-007).

**Alternativas.** Cloud Run ou servidor próprio: mais controle, mais operação. Fila + worker
na nuvem: necessário só com volume bem maior.

**Consequências.** + Deploy pelo GitHub, preview por PR, zero servidor para cuidar. − Limite de
tempo por mensagem; nada pode rodar "depois" dentro da função.

## ADR-011: Postgres (Supabase) como fonte da verdade, inclusive da conversa

**Contexto.** Função sem estado (ADR-010) e uma pergunta aberta que pode ser respondida minutos
depois. Fila do PC, avisos, estoque e registros também precisam de um lugar só.

**Decisão.** Supabase Postgres para tudo, com o `PostgresSaver` do LangGraph guardando a conversa
por `loja:telefone`. Migrações numeradas em `banco/`. `loja_id` em todas as tabelas.

**Consequências.** + Um banco só para operar e consultar; relatório = SQL. + Pronto para várias
lojas. − Tokens da Shopee ainda ficam no Firestore (herança; migrar é opcional).

## ADR-012: Modelos em fila com reserva, provedor isolado numa função

**Contexto.** O provedor de IA pode responder sobrecarga (503) ou limite de chamadas (429).
Cada papel precisa de um tamanho diferente: classificar é mais simples que escolher argumentos.

**Decisão.** `agente.modelo()` monta o modelo com uma fila de reservas (`with_fallbacks`): se um
falha, a mesma chamada vai para o próximo. O orquestrador usa modelos leves; os especialistas,
modelos maiores. As listas vêm de variáveis de ambiente (`GEMINI_MODELOS`,
`GEMINI_MODELOS_ORQUESTRADOR`). As evals usam uma chave separada.

**Consequências.** + Uma falha do provedor raramente vira mensagem sem resposta. + Trocar de
modelo ou provedor mexe em um lugar só, e as evals dizem se piorou. − Um modelo reserva pode
escolher diferente do principal (para medir, rodar as evals com a lista trocada).
