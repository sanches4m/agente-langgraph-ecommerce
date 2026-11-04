# Grafo do agente (LangGraph)

Desenho gerado do próprio código (`agente.py`) com
`agente.construtor.compile().get_graph().draw_mermaid()`. O GitHub mostra como fluxograma.

- Linha cheia: caminho fixo. Linha pontilhada: caminho decidido na hora (o orquestrador escolhe o
  especialista; o especialista decide se usa ferramenta ou responde).
- Cada especialista só enxerga as ferramentas da área dele. Ações (emitir nota, preparar envio,
  imprimir etiqueta) pausam o grafo e pedem confirmação no WhatsApp (botões Sim/Não) antes de agir.
- `resposta_pronta`: ações e relatórios terminam com o texto pronto de `textos.py`, sem a IA reescrever.

```mermaid
---
config:
  flowchart:
    curve: linear
---
graph TD;
	__start__([<p>__start__</p>]):::first
	orquestrador(orquestrador)
	atendimento(atendimento)
	shopee(shopee)
	ferramentas_shopee(ferramentas_shopee)
	notas_fiscais(notas_fiscais)
	ferramentas_notas(ferramentas_notas)
	envios(envios)
	ferramentas_envios(ferramentas_envios)
	relatorios(relatorios)
	ferramentas_relatorios(ferramentas_relatorios)
	resposta_pronta(resposta_pronta)
	__end__([<p>__end__</p>]):::last
	__start__ --> orquestrador;
	envios -.-> __end__;
	envios -. &nbsp;tools&nbsp; .-> ferramentas_envios;
	ferramentas_envios -.-> envios;
	ferramentas_envios -.-> resposta_pronta;
	ferramentas_notas -.-> notas_fiscais;
	ferramentas_notas -.-> resposta_pronta;
	ferramentas_relatorios -.-> relatorios;
	ferramentas_relatorios -.-> resposta_pronta;
	ferramentas_shopee --> shopee;
	notas_fiscais -.-> __end__;
	notas_fiscais -. &nbsp;tools&nbsp; .-> ferramentas_notas;
	orquestrador -.-> atendimento;
	orquestrador -.-> envios;
	orquestrador -.-> notas_fiscais;
	orquestrador -.-> relatorios;
	orquestrador -.-> shopee;
	relatorios -.-> __end__;
	relatorios -. &nbsp;tools&nbsp; .-> ferramentas_relatorios;
	shopee -.-> __end__;
	shopee -. &nbsp;tools&nbsp; .-> ferramentas_shopee;
	atendimento --> __end__;
	resposta_pronta --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
```
