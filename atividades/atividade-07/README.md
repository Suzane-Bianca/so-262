Suzane Bianca da Silva

Explicação dos testes propostos:

Teste 1 (Requisição do Processo P1 - Solicitado: [1, 0, 2]):
O processo P1 fez uma requisição de recursos que não ultrapassava a sua necessidade máxima nem os recursos disponíveis no momento. O algoritmo realizou uma alocação provisória e, ao testar o novo estado do sistema, confirmou que este permanecia num estado seguro (ou seja, existia uma sequência de execução que evitava o deadlock). Como resultado, a requisição foi concedida com sucesso e os recursos foram alocados ao processo P1.

Teste 2 (Requisição do Processo P4 - Solicitado: [3, 3, 0]):
O processo P4 solicitou uma quantidade de recursos que, após a simulação de alocação provisória, deixou o sistema num estado inseguro. Isto significa que os recursos restantes não seriam suficientes para garantir que os outros processos conseguissem terminar a sua execução, criando um risco real de deadlock. Por precaução, o algoritmo efetuou um rollback (reverteu a alocação provisória), a requisição foi negada e o processo P4 foi forçado a aguardar.
