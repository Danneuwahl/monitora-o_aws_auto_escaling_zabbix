# AWS Auto Scaling Groups (ASG) Monitoring with Zabbix 7.0

Este projeto fornece uma solução completa para monitorar múltiplos **Auto Scaling Groups (ASG)** da AWS utilizando o Zabbix Agent. A solução utiliza Descoberta de Baixo Nível (LLD) para separar os dados por grupo, trata duplicidades de registros no JSON e converte status de texto em dados numéricos para geração de gráficos.

## 🚀 Funcionalidades

* **Descoberta Automática (LLD):** Identifica dinamicamente cada ASG Group no arquivo de log.
* **Tratamento de Duplicidade:** Script JavaScript integrado para processar apenas uma entrada por grupo, evitando erros de chaves duplicadas.
* **Gráficos Dinâmicos:** Conversão de status (`Successful`/`Failed`) para valores binários (`1`/`0`).
* **Eficiência de Dados:** Uso de Itens Dependentes para ler o arquivo apenas uma vez por ciclo de coleta.

## 🛠️ Configuração no Servidor (Rocky Linux 9)

No servidor onde reside o arquivo de log da AWS, siga os passos:

1.  **Localize o arquivo de dados:** Certifique-se de que o arquivo esteja em `/home/daniel/teste.txt`.
2.  **Configure o UserParameter:**
    Crie ou edite o arquivo `/etc/zabbix/zabbix_agentd.d/aws_asg.conf`:
    ```bash
    UserParameter=aws.asg.discovery,cat /home/daniel/teste.txt
    ```
3.  **Reinicie o Agente:**
    ```bash
    sudo systemctl restart zabbix-agent
    ```

## ⚙️ Configuração do Template (Zabbix Web)

### 1. Tratamento de Unicidade (LLD)
Para evitar o erro de itens duplicados, a **Discovery Rule** utiliza o seguinte processamento JavaScript na aba **Preprocessing**:

```javascript
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

2. Conversão para Gráficos
Para que o Zabbix gere gráficos, o status é convertido em valores binários:

Successful ➔ 1

Failed ➔ 0

Mapeamento de Valor (Value Mapping):
| Valor | Texto |
| :--- | :--- |
| 1 | Successful |
| 0 | Failed |

📊 Como Visualizar os Gráficos
Após a descoberta, vá em Monitoring > Latest Data.

Filtre pelo host e procure por ASG [Nome do Grupo]: Last Event Status.

Os gráficos automáticos estarão disponíveis em Graph Prototypes dentro do Host ou através de Widgets de Dashboard.

📥 Importação do Template
No menu lateral do Zabbix, vá em Data collection > Templates.

Clique em Import e selecione o arquivo .yaml disponibilizado neste repositório.

Certifique-se de que a versão do seu Zabbix é 7.0 LTS ou superior.

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
          value_type: TEXT

Desenvolvido por: Daniel
