# Arquitetura do CS77 human access

## 1. Objetivo e âmbito

O CS77 human access é uma demonstração experimental de verificação de humanidade com World ID 4.0 e aplicação do princípio **One Human, One Claim**. O objetivo é conceder acesso a um recurso digital a uma pessoa verificada e impedir que a mesma pessoa obtenha mais do que uma reclamação de acesso.

A arquitetura abrange três componentes independentes, as respetivas APIs e os dados mínimos necessários à deduplicação e às métricas. Não abrange a divulgação de identidades pessoais, a criação de um sistema de identidade próprio, nem funcionalidades de produto que não sejam necessárias para a demonstração. O projeto não é um produto oficial World ID.

## 2. Arquitetura de repositórios e deployments

O sistema é composto por três repositórios GitHub e três projetos Vercel independentes:

| Repositório | Projeto Vercel | Função principal |
|---|---|---|
| `human-verification` | Projeto próprio | Verificação World ID 4.0, deduplicação e decisão de acesso |
| `cs77-human-interface` | Projeto próprio | Interface e experiência de acesso |
| `cs77-human-counter` | Projeto próprio | Apresentação de métricas públicas agregadas |

Não será utilizada arquitetura monorepo. Os componentes não partilham código-fonte, diretórios, deployments ou configurações de execução. A integração ocorre através de APIs HTTP versionadas e documentadas.

## 3. Responsabilidades e limites

### 3.1 `human-verification`

É o serviço de autoridade para a verificação e a concessão de acesso. Deve:

- iniciar ou coordenar o fluxo World ID 4.0;
- validar provas no servidor, com a configuração da aplicação e a ação esperada;
- aplicar a regra de reclamação única de forma atómica;
- emitir e validar sessões de acesso temporárias;
- manter os registos mínimos de reclamações e os agregados necessários às métricas;
- expor APIs versionadas para a interface e o contador.

É o único componente que acede à base de dados de reclamações e aos segredos de verificação. Não deve persistir provas World ID nem dados biométricos ou de identidade.

### 3.2 `cs77-human-interface`

É a camada de apresentação e experiência do utilizador. Deve:

- explicar a demonstração e permitir iniciar a verificação;
- apresentar os estados de verificação, acesso concedido, reclamação repetida e falha;
- apresentar o recurso protegido apenas quando a concessão de acesso for confirmada por `human-verification`.

A interface não valida provas, não decide se uma reclamação é nova, não acede à base de dados e não guarda segredos World ID. Um estado enviado apenas pelo navegador nunca é prova suficiente de acesso concedido.

### 3.3 `cs77-human-counter`

É uma apresentação pública de contagens agregadas. Deve:

- consultar o endpoint público de métricas de `human-verification`;
- mostrar exclusivamente `UNIQUE HUMANS` e `REPEATED CLAIM ATTEMPTS BLOCKED`;
- tratar a indisponibilidade da API sem inventar ou manter valores como se fossem atuais.

Não acede diretamente à base de dados, não recebe identificadores por pessoa e não participa na decisão de acesso.

## 4. Fluxo de verificação World ID 4.0

1. A pessoa abre `cs77-human-interface` e inicia a verificação.
2. A interface solicita a `human-verification` o início de um pedido associado à aplicação e à ação configuradas.
3. A pessoa completa o fluxo World ID 4.0 através da integração suportada.
4. A prova é submetida ao backend de `human-verification`, que valida a prova e o respetivo contexto. A validação no cliente não concede acesso.
5. Após validação bem-sucedida, o serviço calcula o identificador pseudónimo de deduplicação e tenta registar a reclamação.
6. Se a reclamação for nova, o serviço concede acesso e emite uma sessão temporária. A interface valida a confirmação e apresenta o recurso.
7. Se já existir uma reclamação para esse identificador, o serviço bloqueia a repetição, atualiza a métrica respetiva e não concede uma nova sessão.
8. `cs77-human-counter` consulta os totais agregados e apresenta as duas métricas públicas definidas neste documento.

Falhas de validação, pedidos malformados e erros de rede não contam como tentativas repetidas bloqueadas. Essa métrica representa uma repetição confirmada após validação bem-sucedida.

## 5. Regra One Human, One Claim

A garantia é aplicada em `human-verification`, na persistência, e não na interface. Após validar a prova, o backend deriva um identificador pseudónimo do nullifier usando HMAC com uma chave secreta do servidor. A chave HMAC não é enviada ao cliente nem guardada no repositório.

A base de dados aplica uma restrição `UNIQUE` ao identificador derivado. A tentativa de inserir a reclamação é atómica; assim, pedidos concorrentes para a mesma pessoa não podem ambos obter uma reclamação. A aplicação não deve depender apenas de uma consulta prévia, de uma sessão no navegador ou de um bloqueio em memória.

- Inserção nova: registar uma reclamação e conceder acesso.
- Conflito com a restrição `UNIQUE`: negar novo acesso e contabilizar uma repetição bloqueada.
- Prova inválida: negar acesso, sem alterar as métricas de repetição.

O HMAC reduz a exposição do nullifier em repouso, mas continua a ser um identificador pseudónimo. Deve ser protegido como dado sensível e retido apenas pelo período necessário à deduplicação da demonstração. A rotação da chave HMAC exige um procedimento de migração controlada dos identificadores de deduplicação existentes, preservando a unicidade das reclamações durante e após a migração.

## 6. Sessões de acesso seguras e temporárias

A concessão de acesso deve ser representada por uma sessão de curta duração, emitida apenas pelo backend após verificação e reclamação nova. Pode ser um identificador opaco aleatório ou um token assinado com expiração curta; a escolha concreta será feita na implementação.

A sessão deve:

- ser transmitida por canal HTTPS e protegida no navegador com atributos seguros, como `HttpOnly`, `Secure` e política `SameSite` apropriada, quando usar cookie;
- ter validade limitada, âmbito restrito ao recurso e mecanismo de revogação ou invalidação quando aplicável;
- não conter prova World ID, nullifier, chave HMAC ou dados pessoais;
- não ser registada em logs nem exposta em URLs, métricas ou respostas públicas;
- ser validada pelo backend nas operações protegidas. A interface não pode criar ou prolongar a sessão por iniciativa própria.

## 7. Métricas públicas

A API pública de métricas expõe apenas estes campos e rótulos:

- **`UNIQUE HUMANS`** — número de reclamações únicas que receberam acesso;
- **`REPEATED CLAIM ATTEMPTS BLOCKED`** — número de tentativas repetidas bloqueadas após validação de uma prova.

Cada repetição confirmada deve ser contabilizada uma única vez, de forma atómica. Reenvios técnicos da mesma operação devem ser idempotentes e não aumentar artificialmente a métrica. As respostas não incluem eventos individuais, identificadores pseudónimos, provas, timestamps por utilizador, endereços de rede nem informação de sessão. As métricas são agregadas no serviço de verificação; o contador limita-se a apresentá-las.

## 8. Privacidade e proteção de dados

- Não guardar provas World ID, imagens, dados biométricos, nomes, emails ou identidade pública.
- Guardar apenas o HMAC do nullifier necessário à deduplicação e os dados mínimos de estado/contagem.
- Guardar chaves de assinatura, segredos HMAC e credenciais apenas em variáveis de ambiente protegidas ou num gestor de segredos do ambiente de execução.
- Nunca versionar segredos nem enviá-los para os projetos de interface ou contador.
- Não incluir provas, nullifiers, identificadores pseudónimos ou tokens em logs, telemetria ou mensagens de erro públicas.
- Aplicar controlo de acesso à base de dados e definir períodos de retenção e eliminação para dados operacionais.
- Publicar apenas os dois totais agregados especificados.

## 9. Comunicação por APIs HTTP versionadas

A comunicação entre os projetos é feita exclusivamente por APIs HTTP documentadas e versionadas, por exemplo sob o prefixo `/api/v1/`. Os contratos devem definir autenticação quando necessária, formatos de pedido e resposta, erros, limites e compatibilidade.

- `cs77-human-interface` comunica com `human-verification` para iniciar a verificação e obter o estado/sessão de acesso.
- `cs77-human-counter` consulta o endpoint público de métricas agregadas de `human-verification`.
- `human-verification` valida origem, contexto e pedidos recebidos; CORS não substitui autenticação nem validação de sessão.

Os consumidores não acedem diretamente à base de dados do serviço de verificação. Alterações incompatíveis às APIs exigem uma nova versão e coordenação documentada entre os repositórios.

## 10. Independência operacional

Cada componente tem repositório, projeto Vercel, domínio, pipeline de publicação, variáveis de ambiente, permissões e ciclo de manutenção próprios. Não há dependência de build entre repositórios nem partilha de credenciais de deploy.

A indisponibilidade ou desativação do contador não deve interromper a verificação ou o acesso. A indisponibilidade da interface não deve corromper reclamações já registadas. A desativação de `human-verification` impede novas verificações e concessões, mas não exige eliminar os outros repositórios ou deployments.

## 11. Papel de `cs77-human-access`

O repositório atual `cs77-human-access` é o repositório de coordenação e documentação do projeto. Regista a arquitetura aprovada, os contratos entre componentes, decisões técnicas, ligações para os três repositórios e orientações operacionais.

Não implementa nem duplica verificação, interface, contador ou APIs de produção. Não é um quarto componente de execução nem um monorepo. A implementação de cada componente pertence exclusivamente ao respetivo repositório.

## 12. Fases recomendadas de implementação

1. **Preparação:** criar/configurar os três repositórios e projetos Vercel independentes; definir domínios, ambientes e responsáveis.
2. **Contratos:** especificar as APIs versionadas, estados, erros, regras de origem e formato das métricas.
3. **Verificação e persistência:** integrar World ID 4.0 em `human-verification`; criar persistência mínima, HMAC e restrição `UNIQUE`; garantir a operação atómica.
4. **Sessões:** implementar emissão, expiração, validação e proteção das sessões de acesso.
5. **Interface:** desenvolver `cs77-human-interface` contra o contrato publicado, cobrindo os estados de sucesso, repetição e falha.
6. **Contador:** desenvolver `cs77-human-counter` usando apenas o endpoint agregado.
7. **Operação:** configurar segredos por projeto, políticas de logs/retensão, monitorização sem dados sensíveis e procedimentos independentes de publicação e desativação.
