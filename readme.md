📑Documentação Técnica: Monitoramento AWS ASG (Zabbix 7.0)

1. Configuração de Coleta (Backend)

Para que o Zabbix diferencie os grupos de Auto Scaling, o Agente deve ler o JSON bruto.

Arquivo de Dados: /home/daniel/teste.txt

UserParameter (Rocky Linux): No arquivo /etc/zabbix/zabbix_agentd.d/aws_asg.conf:

Bash

UserParameter=aws.asg.discovery,cat /home/daniel/teste.txt

Aplicação: sudo systemctl restart zabbix-agent

2. Ajuste de Grupos na Regra LLD (Tratamento de Duplicidade)

Quando o arquivo contém múltiplas atividades para o mesmo grupo, o Zabbix gera erro de "item já existe". Para separar por nome de grupo de forma única:

Pré-processamento da Regra LLD: Adicione um passo JavaScript na regra de descoberta:

JavaScript
var data = JSON.parse(value);
var uniqueNames = {};
var result = [];
for (var i = 0; i < data.length; i++) {
    var name = data[i].AutoScalingGroupName;
    if (!uniqueNames[name]) {
        uniqueNames[name] = true;
        result.push(data[i]);
    }
}
return JSON.stringify(result);

Mapeamento de Macro: No campo LLD Macro Paths, associe {#ASG_NAME} ao caminho $.AutoScalingGroupName.

3. Conversão de Dados para Gráficos

O Zabbix não gera gráficos de linha para textos. Convertemos o status para valores numéricos:

Value Mapping: Crie um mapeamento chamado AWS ASG Status:

1 → Successful

0 → Failed

JavaScript no Item Prototype: No pré-processamento do item de status, adicione:

JavaScript

return (value === 'Successful') ? 1 : 0;
Tipo de Informação: Altere o protótipo para Numeric (unsigned).

4. Criação de Graph Prototypes (Passo a Passo)
Com os itens convertidos para números, siga estes passos para gerar os gráficos automáticos:

Acesse Data collection > Templates e abra o template.

Entre em Discovery rules e clique em Graph prototypes.

Clique em Create graph prototype.

Configurações:

Name: Status de Escalonamento: {#ASG_NAME}

Width/Height: 900x200 (recomendado).

Adicionar Itens:

Clique em Add.

Na janela de seleção, clique no link Item prototypes no topo direito.

Selecione ASG [{#ASG_NAME}]: Last Event Status.

Salvar: Clique em Add para finalizar. O Zabbix criará um gráfico para cada grupo novo descoberto.


5. YAML do Template Completo (Zabbix 7.0)

zabbix_export:
  version: '7.0'
  template_groups:
    - uuid: 7f83353840664190a7960306114a7065
      name: 'Templates/Cloud'
  templates:
    - uuid: 550e8400e29b41d4a716446655440000
      template: 'AWS Auto Scaling by Group Discovery'
      name: 'AWS Auto Scaling by Group Discovery'
      groups:
        - name: 'Templates/Cloud'
      items:
        - uuid: 9b20727c3677495287001c3127885994
          name: 'ASG: Get Raw Data'
          key: aws.asg.discovery
          history: 1d
          trends: '0'
          value_type: TEXT
      discovery_rules:
        - uuid: f47ac10b58cc4372a5670e02b2c3d479
          name: 'ASG Group Discovery'
          type: DEPENDENT
          key: aws.asg.groups.discovery
          master_item:
            key: aws.asg.discovery
          lld_macro_paths:
            - lld_macro: '{#ASG_NAME}'
              path: '$.AutoScalingGroupName'
          preprocessing:
            - type: JSONPATH
              parameters:
                - '$.Activities'
            - type: JAVASCRIPT
              parameters:
                - |
                  var data = JSON.parse(value);
                  var uniqueNames = {};
                  var result = [];
                  for (var i = 0; i < data.length; i++) {
                      var name = data[i].AutoScalingGroupName;
                      if (!uniqueNames[name]) {
                          uniqueNames[name] = true;
                          result.push(data[i]);
                      }
                  }
                  return JSON.stringify(result);
          item_prototypes:
            - uuid: 2f0e8445312a4f51872e06778f697412
              name: 'ASG [{#ASG_NAME}]: Last Event Status'
              type: DEPENDENT
              key: 'aws.asg.status[{#ASG_NAME}]'
              master_item:
                key: aws.asg.discovery
              value_type: UINT64
              valuemap:
                name: 'AWS ASG Status'
              preprocessing:
                - type: JSONPATH
                  parameters:
                    - '$.Activities[?(@.AutoScalingGroupName == "{#ASG_NAME}")].StatusCode.first()'
                - type: JAVASCRIPT
                  parameters:
                    - "return (value === 'Successful') ? 1 : 0;"
                - type: DISCARD_UNCHANGED_HEARTBEAT
                  parameters:
                    - 1h
          graph_prototypes:
            - uuid: a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
              name: 'Status de Escalonamento: {#ASG_NAME}'
              graph_items:
                - color: 1A7C11
                  item:
                    host: 'AWS Auto Scaling by Group Discovery'
                    key: 'aws.asg.status[{#ASG_NAME}]'
  value_maps:
    - uuid: b1c2d3e4f5g6h7i8j9k0l1m2n3o4p5q6
      name: 'AWS ASG Status'
      mappings:
        - value: '1'
          newvalue: Successful
        - value: '0'
          newvalue: Failed
