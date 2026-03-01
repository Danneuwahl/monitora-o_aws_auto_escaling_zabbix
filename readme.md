# AWS Auto Scaling Groups (ASG) Monitoring with Zabbix 7.0

Este projeto oferece uma solução robusta para monitorar as atividades de múltiplos **Auto Scaling Groups (ASG)** da AWS através de um Zabbix Agent rodando em Rocky Linux 9. A solução utiliza Descoberta de Baixo Nível (LLD) para criar itens e gráficos automaticamente para cada grupo identificado.

## 🚀 Funcionalidades

- **Descoberta Automática (LLD):** Identifica dinamicamente novos grupos de ASG.
- **Deduplicação de Dados:** Script JavaScript integrado para garantir que apenas um item seja criado por grupo, mesmo com múltiplas entradas no JSON.
- **Gráficos de Status:** Conversão de status textual para numérico, permitindo a visualização de histórico em gráficos.
- **Eficiência:** Itens dependentes reduzem a carga no servidor, lendo o arquivo de origem apenas uma vez por ciclo.

## 🛠️ Configuração no Servidor (Rocky Linux 9)

O Zabbix Agent precisa de acesso ao arquivo JSON gerado pela AWS (ou simulado).

1. **Arquivo de Dados:** Certifique-se de que o arquivo está no caminho `/home/daniel/teste.txt`.
2. **UserParameter:** Adicione a seguinte linha ao arquivo `/etc/zabbix/zabbix_agentd.d/aws_asg.conf`:
   ```bash
   UserParameter=aws.asg.discovery,cat /home/daniel/teste.txt

   Reinicialização: ```bash
3- sudo systemctl restart zabbix-agent

⚙️ Configuração do Zabbix (Template)
1. Regra de Descoberta (LLD)
A regra utiliza um filtro JavaScript para extrair nomes únicos de grupos do campo AutoScalingGroupName.

Script de Pré-processamento:

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

📄 Exemplo de Estrutura do Arquivo JSON (teste.txt)

{
    "Activities": [
        {
            "AutoScalingGroupName": "Producao-WebService",
            "StatusCode": "Successful",
            "Description": "Terminating instance: i-0abc123"
        }
    ]
}

Desenvolvido por: Daniel
