# Chatbot de Atendimento ao Cliente Integrado com AWS Step Functions, Amazon Bedrock e DynamoDB

## Estudo de Caso: Aplicação como Chatbot para Atendimento ao Cliente em Chamados via Site

### Introdução

Este documento apresenta um estudo de caso que demonstra como implementar um chatbot de atendimento ao cliente utilizando AWS Step Functions, Amazon Bedrock e Amazon DynamoDB. O objetivo é fornecer um sistema automatizado que interaja com os clientes de forma inteligente, respondendo a chamados recebidos via site e armazenados em um banco de dados.

### Cenário

Uma empresa recebe diariamente um grande volume de chamados de suporte ao cliente através de seu site. Cada chamado contém perguntas, reclamações ou solicitações que são armazenadas em uma tabela do Amazon DynamoDB para processamento posterior. O atendimento manual desses chamados é demorado e consome muitos recursos humanos.

### Solução Proposta

Implementar um **chatbot inteligente** que:

- **Leia os chamados armazenados no DynamoDB**: O sistema captura cada chamado enviado pelos clientes.
- **Processe cada chamado utilizando um modelo de linguagem avançado via Amazon Bedrock**: O modelo gera respostas personalizadas e contextuais.
- **Mantenha um histórico de conversa**: As interações anteriores são consideradas para fornecer respostas mais relevantes.
- **Trunque o histórico de conversa quando necessário**: Gerencia o tamanho do histórico para evitar limitações técnicas.
- **Orquestre todo o fluxo de trabalho com AWS Step Functions**: Coordena as etapas de forma eficiente e escalável.

### Benefícios da Solução

- **Atendimento Automatizado**: Reduz a necessidade de intervenção humana, acelerando o tempo de resposta.
- **Respostas Personalizadas**: O uso de IA permite respostas mais relevantes e contextuais.
- **Escalabilidade**: Capaz de lidar com um grande volume de chamados simultaneamente.
- **Eficiência Operacional**: Libera a equipe para focar em questões mais complexas que exigem intervenção humana.

---

## Explicação do Fluxo de Execução

### Visão Geral

A máquina de estados do AWS Step Functions coordena o processo de:

1. **Inicialização**: Obtenção da lista de IDs de chamados a serem processados.
2. **Processamento em Loop**: Para cada chamado, realiza uma série de etapas até que todos sejam processados.
3. **Finalização**: Encerra a execução após o processamento completo.

### Detalhamento das Etapas

#### 1. Seed the DynamoDB Table (Inicialização da Tabela DynamoDB)

- **Descrição**: Invoca uma função Lambda (`MyLambdaFunction`) que retorna uma lista de IDs de chamados a serem processados, incluindo `"DONE"` como indicador de término.
- **Resultado**: Armazena a lista em `$.List`.
- **Próximo Estado**: `Initialize Conversation History`.

#### 2. Initialize Conversation History (Inicializar Histórico de Conversa)

- **Descrição**: Inicializa o histórico da conversa como uma string vazia em `$.ConversationHistory`.
- **Próximo Estado**: `For Loop Condition`.

#### 3. For Loop Condition (Condição do Loop)

- **Descrição**: Verifica se o primeiro elemento da lista (`$.List[0]`) é diferente de `"DONE"`.
  - **Se for diferente**: Continua para `Read Next Message from DynamoDB`.
  - **Se for `"DONE"`**: Vai para o estado `Succeed` e encerra a execução.
- **Próximo Estado**: Dependente da condição.

#### 4. Read Next Message from DynamoDB (Ler Próxima Mensagem do DynamoDB)

- **Descrição**: Lê o chamado correspondente ao `MessageId` atual da tabela DynamoDB.
- **Parâmetros**:
  - `TableName`: Nome da tabela DynamoDB contendo os chamados.
  - `Key`: Chave primária para acessar o item (`MessageId`).
- **Resultado**: Armazena o chamado em `$.DynamoDB`.
- **Próximo Estado**: `Update Conversation History with Message`.

#### 5. Update Conversation History with Message (Atualizar Histórico com Mensagem)

- **Descrição**: Adiciona o chamado atual ao histórico da conversa.
- **Parâmetros**:
  - Concatena `$.ConversationHistory` com `$.DynamoDB.Item.Message.S`.
- **Resultado**: Atualiza `$.ConversationHistory`.
- **Próximo Estado**: `Truncate Conversation History`.

#### 6. Truncate Conversation History (Truncar Histórico da Conversa)

- **Descrição**: Invoca a função Lambda `TruncateHistoryFunction` para truncar o histórico se exceder um tamanho máximo (por exemplo, 2000 caracteres).
- **Parâmetros**:
  - `ConversationHistory`: Histórico atual da conversa.
  - `MaxLength`: Comprimento máximo permitido.
- **Resultado**: Recebe o histórico truncado e atualiza `$.ConversationHistory`.
- **Próximo Estado**: `Invoke Model with Message`.

#### 7. Invoke Model with Message (Invocar Modelo com Mensagem)

- **Descrição**: Invoca o modelo de linguagem via Amazon Bedrock, passando o histórico da conversa como prompt.
- **Parâmetros**:
  - `ModelId`: Identificador do modelo (por exemplo, `cohere.command-text-v14`).
  - `Body`:
    - `prompt`: O histórico da conversa (`$.ConversationHistory`).
    - `max_tokens`: Limite de tokens na resposta (por exemplo, 250).
- **Resultado**: Armazena a resposta do modelo em `$.ModelResult`.
- **Próximo Estado**: `Update Conversation History with Response`.

#### 8. Update Conversation History with Response (Atualizar Histórico com Resposta)

- **Descrição**: Adiciona a resposta do modelo ao histórico da conversa.
- **Parâmetros**:
  - Concatena `$.ConversationHistory` com `$.ModelResult.ModelResponse`.
- **Resultado**: Atualiza `$.ConversationHistory`.
- **Próximo Estado**: `Truncate Conversation History After Response`.

#### 9. Truncate Conversation History After Response (Truncar Histórico Após Resposta)

- **Descrição**: Invoca novamente a função Lambda `TruncateHistoryFunction` para truncar o histórico atualizado.
- **Parâmetros**:
  - Mesmos que anteriormente.
- **Resultado**: Atualiza `$.ConversationHistory`.
- **Próximo Estado**: `Pop Element from List`.

#### 10. Pop Element from List (Remover Elemento da Lista)

- **Descrição**: Remove o `MessageId` processado da lista.
- **Parâmetros**:
  - Atualiza `$.List` para `$.List[1:]`.
- **Próximo Estado**: Retorna ao estado `For Loop Condition`.

#### 11. Succeed (Sucesso)

- **Descrição**: Indica que a execução da máquina de estados foi concluída com sucesso.

### Fluxo Visual Simplificado

1. **Início**:
   - Obter lista de chamados (`$.List`).
   - Inicializar histórico de conversa (`$.ConversationHistory`).

2. **Loop** (para cada chamado):
   - Ler próximo chamado do DynamoDB.
   - Atualizar histórico com a mensagem do chamado.
   - Truncar histórico se necessário.
   - Invocar modelo com o histórico.
   - Atualizar histórico com a resposta do modelo.
   - Truncar histórico novamente se necessário.
   - Remover chamado processado da lista.

3. **Término**:
   - Quando todos os chamados forem processados (`"DONE"`), a execução termina com sucesso.

### Considerações Técnicas

- **Funções Lambda**:
  - `MyLambdaFunction`: Retorna a lista de IDs de chamados.
  - `TruncateHistoryFunction`: Implementa a lógica de truncamento do histórico.
- **Amazon DynamoDB**:
  - Armazena os chamados recebidos via site.
- **Amazon Bedrock**:
  - Fornece o modelo de linguagem para gerar respostas inteligentes.
- **AWS Step Functions**:
  - Orquestra todo o fluxo de trabalho de forma escalável e gerenciável.

### Benefícios Operacionais

- **Automação do Atendimento**: Respostas rápidas e automatizadas aos clientes.
- **Melhoria na Satisfação do Cliente**: Respostas contextuais e relevantes aumentam a satisfação.
- **Eficiência**: Redução de custos operacionais com atendimento.
- **Escalabilidade**: Capacidade de lidar com volumes crescentes de chamados sem perda de desempenho.

---

## Como Implementar

1. **Configurar o DynamoDB**:
   - Criar uma tabela para armazenar os chamados dos clientes.
2. **Desenvolver as Funções Lambda**:
   - `MyLambdaFunction`: Para obter a lista de IDs de chamados.
   - `TruncateHistoryFunction`: Para truncar o histórico da conversa.
3. **Configurar o Amazon Bedrock**:
   - Selecionar e configurar o modelo de linguagem apropriado.
4. **Definir a Máquina de Estados no AWS Step Functions**:
   - Utilizar o código JSON fornecido, ajustando os nomes das funções e tabelas conforme necessário.
5. **Testar o Fluxo**:
   - Realizar testes com chamados reais ou simulados para validar o comportamento do chatbot.
6. **Monitorar e Ajustar**:
   - Utilizar as ferramentas de monitoramento da AWS para acompanhar a performance e fazer ajustes conforme necessário.

---

## Conclusão

Este estudo de caso demonstra uma solução eficaz para automatizar o atendimento ao cliente utilizando serviços gerenciados da AWS. Ao integrar o AWS Step Functions, Amazon Bedrock e Amazon DynamoDB, é possível criar um chatbot inteligente que melhora a eficiência operacional e a satisfação dos clientes.

---

**Nota**: Certifique-se de que todos os serviços AWS utilizados estejam corretamente configurados e que as permissões necessárias estejam definidas para que a máquina de estados possa acessar os recursos necessários.
