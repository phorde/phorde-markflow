# PRD — MarkFlow

**Atualizado:** 2026-10-22  
**Estado para produto web:** Rascunho. O núcleo CLI/OCR é maduro; o fluxo web ainda não cumpre o percurso real de envio e processamento de um PDF.  
**Repositório:** phorde/markflow  
**Decisão de escopo:** este documento registra o estado e os requisitos do produto. Não autoriza publicação, cadastro no Hub, renomeação de repositório ou migração de hospedagem.

## 1. Problema e valor

MarkFlow converte PDFs de qualidade variada em Markdown estruturado, HTML sanitizado e relatórios JSON auditáveis. O produto deve tornar explícito de onde veio cada trecho, qual método/modelo foi usado, a confiança por página e se a extração está pronta para exportação.

O valor central registrado em .planning/PROJECT.md é: um documento nunca pode ser apresentado como processado com segurança sem que o estado da extração, a confiança e as verificações de política sustentem essa conclusão. MarkFlow auxilia extração e revisão; não valida conteúdo clínico nem substitui julgamento profissional.

## 2. Usuários e objetivos

- Pessoa que processa localmente PDFs individuais ou lotes e escolhe custo, velocidade, qualidade ou execução totalmente local.
- Pessoa que precisa inspecionar páginas de baixa confiança, corrigir o Markdown, solicitar reprocessamento e exportar um resultado com trilha de auditoria.
- Operador que configura provedores e modelos uma vez, entende as recomendações e mantém credenciais fora dos arquivos gerados.

Sucesso significa que o mesmo documento produz estados e relatórios coerentes, que dúvidas de extração ficam visíveis e que a interface nunca marca como pronto um trabalho incompleto ou não revisado segundo a política escolhida.

## 3. Escopo funcional

### Núcleo CLI e TUI

1. Aceitar um PDF ou diretório de PDFs e produzir artefatos por documento.
2. Extrair texto nativo quando adequado e acionar OCR local ou remoto conforme qualidade, confiança e configuração.
3. Oferecer EasyOCR, RapidOCR e Tesseract como engines locais suportadas, normalizando confiança para o intervalo de 0 a 1.
4. Integrar provedores OpenAI-compatible, Anthropic, Gemini, OpenRouter, Z.AI e endpoints compatíveis configurados pelo usuário.
5. Descobrir modelos disponíveis, comparar sinais do OCRBench v2 e selecionar/rankear candidatos de forma determinística e explicável.
6. Oferecer perfis auto, fast, quality, local e remote; e objetivos fast, balanced e high-accuracy-ocr.
7. Gerar os artefatos convencionados: *.canonical.md, *.canonical.html e *.canonical.report.json. O HTML deve ser sanitizado.
8. Guiar configuração de credenciais, provedor, descoberta de modelos, benchmark e perfil pela TUI; a entrada de chaves deve ser mascarada.

### Aplicação web/API

1. Expor criação e consulta de jobs, estado de páginas, eventos de progresso, revisão, pedidos de reprocessamento e exportação.
2. Mostrar estado, confiança, origem, avisos, tempo, decisões de roteamento, revisão e ciclo de vida dos artefatos por job.
3. Permitir correção humana do Markdown e aprovação explícita de páginas de baixa confiança antes da exportação.
4. Ingerir o arquivo PDF real de forma autenticada, limitada e rastreável. A interface e a API atuais criam jobs usando metadados como nome e número de páginas; o próprio frontend declara que o upload binário ainda não foi implementado.
5. Executar a pipeline real no worker e disponibilizar os artefatos resultantes. O worker atual emite um ciclo de eventos sintético e uma URI job://; ele ainda não chama a pipeline OCR nem produz o resultado do documento.

## 4. Regras de negócio e segurança

- Um trabalho só pode ser exportado quando estiver concluído e marcado como pronto após as validações e a revisão exigidas pelo perfil.
- Em modo quality/medical_strict, baixa confiança, revisão ausente, resultado incompleto ou necessidade de reprocessamento devem falhar fechado; não converter ausência de evidência em aprovação.
- Modo local não faz chamadas remotas a LLM. Testes padrão não fazem chamadas pagas nem dependem de rede ou binários OCR instalados.
- Chaves são de escopo de requisição/memória: nunca persistidas em banco, cache, eventos Redis, estado canônico, logs, relatórios, dead letters ou arquivos de saída. Nunca aceitá-las como argumento de CLI.
- URLs remotas com autenticação exigem HTTPS; HTTP é permitido apenas em localhost para desenvolvimento. Redirecionamentos autenticados devem ser bloqueados para evitar encaminhamento de credenciais.
- A revisão e os artefatos de cada job devem ser acessíveis somente ao proprietário autorizado daquele job. CORS sozinho não constitui autorização.
- Eventos podem ser entregues ao menos uma vez; o consumidor deve deduplicar por event_id e só confirmar mensagens após a atualização do estado canônico.
- Documentos sensíveis não podem entrar em cache persistente por padrão no fluxo estrito. Não incluir documentos pessoais ou clínicos reais em fixtures ou no GitHub.

## 5. Requisitos não funcionais

- Segurança e privacidade por padrão, com credenciais redigidas, upload limitado, isolamento entre usuários/jobs e política explícita de expiração e remoção de artefatos.
- Auditabilidade: relatório final com status do documento e páginas, confiança, avisos, tempo, origem, cache, revisão e contexto de roteamento.
- Confiabilidade: timeout explícito, retries limitados, progresso terminal, reprocessamento idempotente e recuperação de mensagens sem corromper o estado final.
- Determinismo: usar fixtures geradas, mocks e stubs para que os testes padrão não precisem de credenciais, pagamentos, rede ou OCR de host.
- Manutenibilidade: Python 3.10+, contratos HTTP/evento versionados, documentação/decisões rastreáveis e fronteiras de serviço validadas.
- Acessibilidade e uso em telas estreitas na interface de revisão e acompanhamento de jobs.

## 6. Arquitetura e decisões registradas

- Manter Python para o núcleo OCR/ML, sem migração de linguagem por convenção genérica de portfólio; preservar compatibilidade dos fluxos existentes app.py e markflow.
- Manter um monorepo com serviços isolados em services/frontend (Next.js), services/api (FastAPI) e services/worker. Compartilhamento é permitido apenas para módulos neutros do domínio markflow; serviços não importam runtime uns dos outros.
- API é a única autoridade do estado canônico. Frontend envia comandos e representa estado/eventos; worker executa e publica observações, sem alterar estado canônico diretamente.
- Redis Streams usa contratos v1 separados para dispatch, progress e result, schemas versionados e sem segredos em payloads. A entrega é pelo menos uma vez e consumidores precisam ser idempotentes.
- A seleção de modelo usa OCRBench v2 e sinais de qualidade como prioridade; custo e latência são objetivos secundários ponderados pelo modo escolhido. Modelos sem evidência de benchmark recebem pontuação conservadora e aviso.
- Manter teste de rastreabilidade entre os 37 requisitos v1, specs, caminhos de implementação, critérios de aceitação e testes executáveis. A meta de cobertura declarada pelo projeto é 100%.
- Não colapsar API, worker e frontend em um único runtime e não compartilhar sessão, credenciais ou dados de outros micro-apps.

## 7. Estado verificado e lacunas

**Implementado no repositório:** pacote CLI/TUI e pipeline de extração; OCR local e integração LLM provider-agnostic; seleção/routing baseado em benchmark; relatórios e sanitização; contratos de job/review/export e eventos; estrutura de API, worker e frontend; política de isolamento; Docker Compose, Dockerfiles, manifesto Render, testes, CI e artefatos GSD.

**Bloqueadores do produto web:** o frontend/API ainda submetem metadados em vez dos bytes do PDF; o worker atual simula progresso/sucesso sem executar a pipeline; artefatos reais ainda precisam percorrer armazenamento, revisão e exportação de ponta a ponta. Os contratos de API também precisam de autenticação/verificação de propriedade em todas as rotas de leitura, revisão, eventos e exportação, além de autenticação do endpoint interno de eventos, antes de qualquer exposição fora de um ambiente privado. O código auditado não mostra uma guarda de autorização nesses handlers.

**Operação:** existe configuração Render no repositório e uma auditoria anterior registra uso do Render, mas a saúde e o comportamento da implantação atual não foram verificados nesta atualização. A publicação final, o domínio e a integração com o Access do Hub permanecem decisões operacionais pendentes; manter o app fora do Hub até que o fluxo web e a autorização estejam validados.

## 8. Checklist de conclusão do produto web

- [x] Núcleo CLI/TUI e processamento local/remoto por modos documentados.
- [x] Contratos e modelos de estado para jobs, páginas, revisão, exportação e eventos definidos.
- [x] Separação API/worker/frontend, schemas v1, testes e gates de CI presentes no repositório.
- [ ] Implementar upload binário real com limites de tamanho/páginas, validação de PDF e proteção contra abuso.
- [ ] Executar a pipeline OCR real a partir do worker e persistir estados/resultados reais, sem eventos de sucesso sintéticos.
- [ ] Armazenar artefatos com expiração/remoção verificável e entregar downloads reais de Markdown/HTML/JSON.
- [ ] Adicionar autenticação owner-only e autorização por job a todas as rotas privadas; autenticar eventos internos do worker; validar CSRF/CORS conforme o mecanismo escolhido.
- [ ] Testar o percurso completo: envio → processamento → evento → revisão/reprocessamento → exportação → expiração/remoção.
- [ ] Confirmar limites de concorrência, timeouts, retries e comportamento após indisponibilidade de Redis, provider ou worker.
- [ ] Rodar novamente no estado atual os gates Python, cobertura, rastreabilidade, fronteiras de serviço e build do frontend.
- [ ] Verificar a implantação real e decidir a hospedagem/domínio e o gate privado antes de integrar ao Hub.

## 9. Fontes de verdade

Este PRD resume o produto; os contratos e critérios técnicos detalhados permanecem em .planning/REQUIREMENTS.md, .planning/specs/features.json, .planning/specs/feature_acceptance_matrix.md, .planning/PROJECT.md, .planning/STATE.md, .planning/ROADMAP.md, .planning/decisions/DECISION_LOG.md e docs/architecture/. Atualize ambos os níveis quando um requisito mudar e mantenha a rastreabilidade GSD válida.
