1. Objetivo do projeto
O objetivo do projeto foi construir um sistema agêntico de QA jurídico, robusto e auditável, para responder a perguntas sobre legislação portuguesa dos jogos e apostas online, garantindo que:
•	As respostas são estritamente baseadas no texto legal;
•	Não há alucinação jurídica;
•	Perguntas fora do domínio são corretamente filtradas;
•	Perguntas de lookup textual (ex.: “qual é a redação da alínea x) do artigo y.º”) são tratadas de forma determinística;
•	Perguntas interpretativas ou temáticas continuam a funcionar corretamente.

2. Visão geral da pipeline (end-to-end)
A pipeline implementada segue um modelo agêntico com LangGraph, integrando LlamaIndex para retrieval e LlangSmith para observabilidade e um LLM apenas onde é estritamente necessário.
Fluxo de alto nível:
1.	Ingestão e preparação dos documentos;
2.	Indexação semântica + lexical;
3.	Domain gating;
4.	Decisão de retrieval;
5.	Retrieval determinístico ou híbrido;
6.	Classificação de relevância (chunk grading);
7.	Geração da resposta grounded;
8.	Reflexão e autocorreção;
9.	Resposta final ao utilizador.

3. Preparação e indexação dos documentos
3.1 Ingestão
•	PDFs são carregados com PyMuPDFReader para evitar corrupção de texto
•	Metadados legais são adicionados manualmente (fonte, versão, jurisdição)


3.2 Limpeza jurídica do texto
Foram removidos:
•	Numeração de páginas (“28 de 59”, “Pág. 36 de 59”);
•	Cabeçalhos repetidos do Diário da República;
•	Quebras artificiais excessivas.
Isto foi essencial para evitar falsos negativos no retrieval e garantir coerência na deteção de artigos e alíneas.
3.3	Chunking por artigo (decisão crítica)
Em vez de chunking arbitrário, o texto foi dividido por artigo, garantindo que cada node corresponde a uma unidade jurídica lógica. Assim, lookup por “Artigo Y.º” é possível e determinístico e, além disso, não se perde contexto normativo relevante.

4. Estratégia de retrieval (um dos pontos mais críticos)
4.1 Índices utilizados
O sistema usa retrieval híbrido, sem LLM no retrieval:
•	Vector retrieval (embeddings multilingual-e5)
•	BM25 lexical retrieval
4.2 Retrieval determinístico por artigo
Se a pergunta contém: “Artigo Y” o sistema ignora embeddings e BM25 e faz lookup direto por: metadata["artigo"] == N.
Isto resolve completamente falhas de embedding em perguntas jurídicas formais e ambiguidade em pedidos de redação exata de determinado artigo ou alínea.
4.3 Retrieval híbrido (fallback)
Para perguntas temáticas (ex.: autoexclusão):
•	Combina resultados do vector retriever e BM25;
•	Normaliza scores;
•	Funde resultados de forma determinística;
•	Sem dependência de OpenAI ou LLMs no retrieval.



5. Domain Gate
Antes de qualquer retrieval, o sistema avalia se a pergunta é:
•	fora_do_dominio → resposta direta conversacional
•	potencialmente_documental → pipeline jurídico completa
Isto evita: Custos desnecessários, respostas jurídicas para perguntas irrelevantes e ruído no grafo

6. Decisão de uso de ferramentas (retrieval_decider)
Um nó LLM decide:
•	Se consegue responder diretamente
•	Ou se precisa de chamar a tool de retrieval
Este nó não responde juridicamente — apenas decide.

7. Grade chunks (avaliação de aplicabilidade)
Este foi um dos maiores desafios do projeto.
7.1 Problema inicial
•	O LLM rejeitava chunks corretos porque:
o	Não “via” explicitamente a alínea pedida;
o	Ou o chunk continha mais texto do que o esperado.
7.2 Solução final (robusta)
Foram introduzidas duas lógicas complementares:
A) Lookup textual (bypass total do grading)
Se a pergunta é do tipo:
•	“Qual é a redação…” ou “O que diz o artigo…”
Então:
•	O grading é ignorado;
•	Todos os chunks do artigo correto são considerados aplicáveis;
•	Se houver alínea, tenta-se localizar;
•	Se não localizar, mantém-se o artigo como aplicável.
B) Grading normal (LLM)
Para perguntas interpretativas:
•	Cada chunk é avaliado como aplicável ou não;
•	Com justificação;
•	Mantém controlo fino sobre contexto usado.

8. Geração da resposta (grounded)
A geração é estritamente condicionada ao contexto selecionado.
Existem dois modos:
8.1	Lookup textual com alínea
Prompt especial que exige transcrição fiel, sem paráfrase e sem conhecimento externo.
Se a alínea não estiver visível o sistema declara explicitamente a limitação e nunca inventa texto.
8.2 Resposta jurídica geral
•	Apenas com base no contexto;
•	Se algo não constar -> o modelo deve dizê-lo explicitamente.

9. Reflection (self-check jurídico)
Antes da resposta final um nó de revisão jurídica valida se todas as afirmações estão suportadas pelo contexto se não estiverem a resposta é corrigida ou substituída por:
“O documento não contém informação suficiente para responder a esta pergunta.”
Este passo elimina:
•	Over-reach do modelo;
•	Inferências não sustentadas;
•	Risco jurídico.

10. Principais desafios enfrentados e resolvidos
10.1 Chunking inadequado
Problema: chunks demasiado pequenos ou arbitrários.
Solução: chunking por artigo.
10.2 Falhas de retrieval em lookup jurídico
Problema: embeddings falhavam em “Artigo Y.º, alínea x)”.
Solução: lookup determinístico por metadata.

10.3 Rejeição indevida de chunks corretos
Problema: grading demasiado agressivo.
Solução: bypass completo para lookup textual.
10.4 Dependência implícita de OpenAI
Problema: BM25 / QueryFusion acionava LLMs internamente.
Solução: hybrid retrieval manual, sem LLM.
10.5 Alucinação em geração
Problema: respostas “bem escritas”, mas não sustentadas.
Solução: reflection node obrigatório, com contexto suportado nos documentos legais.


