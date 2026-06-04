# Splunk

Este repositório consolida o ecossistema do Splunk Enterprise e Splunk Cloud, detalhando desde a navegação na interface inicial até a construção de consultas complexas em SPL (*Search Processing Language*), gerenciamento de objetos de conhecimento e criação de painéis analíticos para segurança defensiva (Blue Team).

---

## 1. Interface Inicial, Apps e Controle de Acesso (Roles)

### O que faz:
Esse ecossistema dita o seu escopo de trabalho e segurança dentro do Splunk. Os **Apps** funcionam como "subprogramas" ou salas temáticas construídas para resolver casos de uso específicos. As **Roles** (Funções) garantem o princípio do privilégio mínimo dentro do ambiente corporativo.

* **Administrator / `sc_admin` (Cloud):** O nível mais alto. Pode instalar aplicativos, ingerir dados brutos e criar objetos de conhecimento globais para todos os usuários.
* **Power:** Pode criar e compartilhar objetos de conhecimento dentro de um app específico e executar buscas em tempo real.
* **User:** O nível mais restritivo. Visualiza apenas seus próprios objetos de conhecimento ou aqueles que foram explicitamente compartilhamos com ele.

### Exemplos Práticos:
* **Cenário de App:** Ao atuar no SOC, você abre o *Splunk App for Enterprise Security*. Ele já traz painéis nativos para monitorar ataques de ransomware, poupando o analista de criar gráficos do zero.
* **Cenário de Roles:**
  * O **Admin** configura o Splunk para receber logs de auditoria da AWS.
  * O **Power User** cria uma busca automatizada para identificar acessos suspeitos na AWS e compartilha com o time.
  * O **User** (Analista Júnior) consome essa busca compartilhada para investigar um incidente, mas não tem permissão para alterar as fontes de dados ou instalar novos apps.

---

## 2. Interface do Search & Reporting App

### O que faz:
É a "página de pesquisa" principal do analista de segurança. Permite realizar o processo de *Threat Hunting* destrinchando os metadados antes mesmo de processar regras complexas de correlação.

* **Splunk Bar:** Barra superior presente em todo o sistema. Usada para alternar entre apps, editar dados da conta, ver mensagens de erro e monitorar o progresso de tarefas.
* **Search Bar & Time Range Picker:** Local para digitar as queries. O seletor de tempo é a ferramenta mais crítica para a performance de busca.
* **Data Summary:** Fornece um mapeamento dos dados indexados dividido em três pilares fundamentais:
  * **Host:** O nome da máquina, endereço IP ou domínio de origem do log.
  * **Source:** O caminho do arquivo físico, porta de rede ou script gerador do evento.
  * **Sourcetype:** A classificação e formato do dado (ex: `linux_secure`, `cisco_wsa_squid`, `access_combined`).
* **Table Views:** Permite que o usuário explore e prepare dados visualmente via interface (UI), sem a necessidade de escrever linhas de código em SPL.
* **Search History:** Histórico interativo que armazena queries passadas, permitindo filtragem por termos ou por períodos de tempo (Hoje, 7 dias, 30 dias).

### Exemplos Práticos:
* **Uso do Time Range Picker:** Ao investigar um site que ficou fora do ar há poucos minutos, alterar o seletor de tempo para *Last 30 minutes* limita o escopo e evita que o Splunk varra meses de logs desnecessários, acelerando o retorno do servidor.
* **Uso do Data Summary:** Para checar se um novo Firewall implantado na filial já está enviando logs para o SIEM, o analista clica em *Data Summary*, acessa a aba *Hosts* e valida se o IP do dispositivo consta na listagem com eventos recentes.

---

## 3. Ciclo de Vida da Busca, Eventos e Jobs

### O que faz:
Cada consulta iniciada gera uma tarefa interna no servidor chamada **Job**. O gerenciamento do ciclo de vida desse Job permite economizar hardware e documentar evidências periciais durante uma resposta a incidentes.

### Guias de Resultados
* **Events:** Exibe os logs brutos em ordem cronológica inversa (mais novos primeiro). Permite aplicar amostragem (*Event Sampling*) e expandir o evento para visualizar todos os campos extraídos.
* **Patterns:** Agrupa e identifica padrões repetitivos nos dados para acelerar diagnósticos visuais.
* **Statistics & Visualization:** Exibe tabelas estruturadas e elementos gráficos. Só é populada se a query utilizar **Comandos Transformadores** (comandos que convertem logs brutos em tabelas de dados).

### 💡 Exemplos Práticos:
* **Gerenciamento de Job (Segundo Plano):** Ao rodar uma busca massiva de *Threat Hunting* para rastrear um IP suspeito nos últimos 90 dias, o analista pode clicar em *Job -> Send to Background*. A busca continua rodando no servidor em segundo plano e o analista pode fechar o navegador sem perder o progresso.
* **Compartilhamento e Exportação:** Após localizar os logs exatos de uma invasão, clicar em *Job -> Share* mantém a tarefa ativa na memória por **7 dias** (o padrão de buscas normais é expirar em **10 minutos**) para que outros investigadores acessem o link. Os eventos também podem ser baixados em formatos como **JSON, CSV, XML ou Raw** para anexar em relatórios de chamados.

---

## 📑 4. Search Processing Language (SPL) e Operadores

### 🔹 O que faz:
O SPL é a linguagem proprietária utilizada para refinar milhões de linhas de texto confuso em respostas cirúrgicas. A filtragem utiliza *pipes* (`|`) que funcionam como uma linha de produção: o resultado da esquerda vira a entrada da direita.

* **Caracteres Curinga:** O asterisco (`*`) faz buscas parciais (ex: `fail*` retornará *failed*, *failure*, *failing*). A busca de termos comuns não diferencia maiúsculas de minúsculas.
* **Expressões Literais:** Termos compostos entre aspas (ex: `"failed password"`) buscam a frase exata e na mesma ordem.

### Ordem de Precedência dos Operadores Lógicos
Ao combinar múltiplos filtros na barra de busca, o Splunk avalia as condições nesta ordem estrita:
1. **Parênteses `( )`:** Forçam a execução prioritária de qualquer instrução contida em seu interior.
2. **NOT:** Inverte ou exclui a condição seguinte (ex: `failed NOT password`).
3. **OR:** Avalia se pelo menos uma das condições é verdadeira.
4. **AND:** Avalia se ambas as condições são verdadeiras (operador padrão aplicado automaticamente pelo Splunk ao omitir o termo entre espaços).

### Os 5 Componentes de uma Query SPL
*Boa prática: sempre escreva os comandos em letras minúsculas.*

```splunk
index=network sourcetype=cisco-wsa_squid usage=Violation | stats count(usage) as Visits by cs_username
