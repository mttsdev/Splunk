# Splunk

Este repositório consolida o ecossistema do Splunk Enterprise e Splunk Cloud, detalhando desde a navegação na interface inicial até a construção de consultas complexas em SPL (*Search Processing Language*).

---

## 1. Interface Inicial, Apps e Controle de Acesso (Roles)

### O que faz:
Esse ecossistema dita o seu escopo de trabalho e segurança dentro do Splunk. Os **Apps** funcionam como "subprogramas" ou salas temáticas construídas para resolver casos de uso específicos. As **Roles** controlam quem pode acessar o quê.

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
* **Uso do Time Range Picker:** Ao investigar um site que ficou fora do ar há poucos minutos, alterar o seletor de tempo para *Last 30 minutes* limita o escopo e evita que o Splunk varra meses de logs.
* **Uso do Data Summary:** Para checar se um novo Firewall implantado na filial já está enviando logs para o SIEM, o analista clica em *Data Summary*, acessa a aba *Hosts* e valida se o IP do dispositivo está registrado.

---

## 3. Ciclo de Vida da Busca, Eventos e Jobs

### O que faz:
Cada consulta iniciada gera uma tarefa interna no servidor chamada **Job**. O gerenciamento do ciclo de vida desse Job permite economizar hardware e documentar evidências periciais durante uma resposta a incidentes.

### Guias de Resultados
* **Events:** Exibe os logs brutos em ordem cronológica inversa (mais novos primeiro). Permite aplicar amostragem (*Event Sampling*) e expandir o evento para visualizar todos os campos extraídos.
* **Patterns:** Agrupa e identifica padrões repetitivos nos dados para acelerar diagnósticos visuais.
* **Statistics & Visualization:** Exibe tabelas estruturadas e elementos gráficos. Só é populada se a query utilizar **Comandos Transformadores** (comandos que convertem logs brutos em tabelas de dados agregados).

### Exemplos Práticos:
* **Gerenciamento de Job (Segundo Plano):** Ao rodar uma busca massiva de *Threat Hunting* para rastrear um IP suspeito nos últimos 90 dias, o analista pode clicar em *Job -> Send to Background*. A busca continua processando enquanto ele navega por outras áreas.
* **Compartilhamento e Exportação:** Após localizar os logs exatos de uma invasão, clicar em *Job -> Share* mantém a tarefa ativa na memória por **7 dias** (o padrão de buscas normais é expirar após algumas horas).

---

## 4. Search Processing Language (SPL) e Operadores

### O que faz:
O SPL é a linguagem proprietária utilizada para refinar milhões de linhas de texto confuso em respostas cirúrgicas. A filtragem utiliza *pipes* (`|`) que funcionam como uma linha de produção: o resultado de um estágio alimenta o próximo.

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
```

**Decomposição dos Componentes:**

1. **Termos de Busca:** `index=network sourcetype=cisco-wsa_squid usage=Violation` — Define a base estrutural e o escopo da consulta.
2. **Comando:** `stats` — Determina a ação direta que deve ser executada sobre os resultados coletados.
3. **Função:** `count()` — Estabelece a operação matemática ou estatística que será aplicada.
4. **Argumento:** `usage` — Define a variável ou campo específico avaliado pela função.
5. **Cláusula:** `as Visits by cs_username` — Comanda a renomeação da coluna de saída e determina o agrupamento/divisão analítica dos dados.

### Exemplos de Queries para Investigação

#### 1. Filtragem com Precedência Lógica

```splunk
index=firewall (action=blocked OR action=dropped) NOT src_ip=192.168.1.5
```

**O que faz:** Captura logs de tráfego que foram explicitamente bloqueados ou descartados pelo firewall, mas remove do resultado final o endereço IP da máquina de testes do administrador (192.168.1.5), eliminando potenciais falsos positivos na triagem.

#### 2. Detecção de Ataque por Força Bruta (Brute Force)

```splunk
index=firewall action=blocked (reason="bad password" OR reason="invalid user")
| stats dc(user) as usuarios_tentados count as total_erros by src_ip
| where total_erros > 50
| rename src_ip as "IP Atacante"
```

**O que faz:** Analisa falhas de autenticação correlacionando erros por senha incorreta ou usuário inválido. Conta a quantidade de usuários únicos que sofreram tentativas (`dc(user)`) e o volume total de erros agregados por IP de origem. Filtra apenas os IPs com mais de 50 ocorrências e formata a tabela final para o analista.

---

## 5. Objetos de Conhecimento (Knowledge Objects)

### O que faz:
Logs brutos costumam ser confusos, massivos e de difícil leitura. Os Objetos de Conhecimento servem para enriquecer, normalizar e catalogar os dados brutos, permitindo que informações puramente técnicas sejam traduzidas em inteligência de segurança inteligível e reutilizável por todo o time do SOC.

| Categoria | Tipos de Objetos | Função Prática & Exemplos |
|-----------|-----------------|---------------------------|
| **Data Interpretation** | Fields, Field Extractions, Calculated Fields | Extrai campos estruturados de logs amorfos.<br>*Exemplo:* Isolar uma string confusa gerada após `usr_id=` e transformá-la no campo limpo `Usuario`. |
| **Data Classification** | Event Types, Transactions | Agrupa e categoriza eventos relacionados sob uma mesma etiqueta de identificação técnica para buscas rápidas. |
| **Data Enrichment** | Lookups, Workflow Actions | Cruza dados locais em tempo real com fontes externas.<br>*Exemplo:* Carregar uma lista CSV de Threat Intel para marcar automaticamente logs com o campo `is_malicious=True`. |
| **Data Normalization** | Tags, Field Aliases | Padroniza nomenclaturas distintas de fabricantes diferentes.<br>*Exemplo:* Mapear o campo `src` (Cisco) e `source_ip` (Checkpoint) para responderem juntos como `src_ip`. |
| **Data Models** | Hierarchically Structured Datasets | Estrutura dados complexos em árvores hierárquicas para que usuários criem tabelas dinâmicas (Pivot Tables) sem usar código. |

---

## 6. Relatórios (Reports) e Dashboards

### O que faz:
Transforma o resultado de buscas técnicas e comandos transformadores em relatórios recorrentes ou painéis visuais (Dashboards) atualizados em tempo real, ideais para o monitoramento contínuo em telas de gerenciamento do SOC.

### Boas Práticas de Nomeação para Relatórios

Para manter ambientes corporativos organizados, escaláveis e auditáveis, adote uma convenção de nomenclatura padronizada baseada na estrutura:

```
[Grupo/Setor]_[TipoDoObjeto]_[DescricaoBreve]
```

**Exemplo de Aplicação:** `Security_Report_FailedSshAttempts`

- **Security:** O time, setor ou destinação final do objeto.
- **Report:** O tipo de Objeto de Conhecimento que foi gerado.
- **FailedSshAttempts:** Descrição direta do conteúdo (Tentativas falhas de SSH).

### Construção de Dashboards

Um Dashboard é uma coleção de painéis analíticos alimentados por buscas salvas ou relatórios rápidos (gerados diretamente a partir de campos da barra lateral, como a função *Top Values*).

**Customização Visual:** A interface interativa permite arrastar e soltar painéis, alterar modos de visualização (gráficos de pizza, linhas, tabelas ou mapas geográficos) e aplicar o Tema Escuro (Dark Theme).

**Edição de Código (XML):** No modo de edição avançado, os analistas podem acessar a guia *Source* para manipular diretamente a estrutura XML do painel, permitindo customizações avançadas de comportamento, cores dinâmicas de alertas e inputs de controle de tempo customizados.

---

## 7. Boas práticas no Splunk

* Letras Minúsculas: Embora o Splunk aceite comandos em maiúsculas, a convenção padrão e recomendada é digitar comandos e funções em minúsculas.
* Filtros de Tempo: Sempre defina o período antes de rodar a busca para economizar processamento.
* Filtros à Esquerda: Quanto mais filtros você colocar antes do primeiro Pipe (|), mais rápido será o resultado, pois o Splunk terá menos dados para "carregar na memória".
