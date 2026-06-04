# Splunk
Esta seção introduz a arquitetura conceitual do Splunk como uma ferramenta de SIEM (*Security Information and Event Management*), detalhando a anatomia do SPL (*Search Processing Language*) para filtragem de logs, a precedência de operadores lógicos e as melhores práticas de otimização para analistas de Blue Team.

### Conceitos Práticos e Comandos Novidades:

* **Pipes (`|`):** Conectores que funcionam como uma linha de produção, enviando o resultado de uma busca ou comando diretamente como entrada para o próximo.
* **Search Terms (Termos de Busca):** Palavras-chave, campos ou expressões booleanas usadas no início da query para filtrar os dados brutos (ex: `index=security`).
* **Commands (Comandos):** Instruções que definem o que fazer com os dados coletados (ex: `stats`, `table`, `eval`). *Boa prática: escrever sempre em letras minúsculas.*
* **Functions (Funções):** Cálculos específicos realizados dentro de comandos (ex: `count()`, `dc()`, `sum()`).
* **Arguments (Argumentos):** Variáveis ou especificações passadas para refinar funções ou comandos (ex: `limit=5`).
* **Clauses (Cláusulas):** Instruções de agrupamento, ordenação ou condição (ex: `by`, `as`).

### Ordem de Precedência de Operadores Lógicos:

1. **Parênteses `( )`:** Forçam a execução prioritária de qualquer instrução contida em seu interior.
2. **NOT:** Inverte a condição lógica imediatamente seguinte.
3. **OR:** Avalia se pelo menos uma das condições estipuladas é verdadeira.
4. **AND:** Avalia se ambas as condições são verdadeiras (operador padrão do Splunk, aplicado automaticamente mesmo quando oculto).

### Objetos de Conhecimento (Knowledge Objects):

* **Fields (Campos Extraídos):** Transformam dados não estruturados de logs brutos em pares de chave/valor legíveis.
* **Tags e Aliases:** Normalizam e simplificam nomes de campos complexos para um padrão inteligível.
* **Lookups:** Cruzam e enriquecem os dados de logs com tabelas externas (ex: listas de IPs maliciosos).
* **Alertas e Dashboards:** Automações visuais e notificações baseadas em queries salvas para monitoramento contínuo.

> 💡 **Dicas de Otimização (Blue Team):**
> * Sempre declare o escopo temporal e os campos `index` e `sourcetype` no início da query.
> * Filtre o máximo de dados possível *antes* do primeiro pipe (`|`) para economizar memória do servidor.

### Exemplos de Uso

```splunk
# 1. Busca Estruturada com Agrupamento e Renomeação (SPL)
index=security sourcetype=linux_secure status=failure
| stats count() as total_falhas by user
| table user total_falhas

# 2. Investigação de Ameaças (Detecção de Brute Force)
index=firewall action=blocked (reason="bad password" OR reason="invalid user")
| stats dc(user) as usuarios_tentados count as total_erros by src_ip
| where total_erros > 50
| rename src_ip as "IP Atacante"
