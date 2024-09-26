# **Visão Geral do Objetivo**

1. **Automatizar o Encerramento do Processo `./o11`**:
   - **Agendado às 4h da manhã**.
   - **Segunda tentativa às 4h03min**.

2. **Monitorar Uso de Memória RAM**:
   - **Encerrar o processo `./o11` se o uso de memória exceder 60%**.
   - **Executar verificação a cada 5 minutos**.

3. **Manter Logs**:
   - **`kill_o11.log`**: Guardar logs por **7 dias** antes de serem limpos.
   - **`kill_o11_memory.log`**: Guardar logs por **48 horas** antes de serem limpos.

4. **Limpeza Automática de Logs**:
   - **Executar diariamente às 00h (meia-noite)** para limpar logs antigos.

---

### **Passo a Passo Completo**

#### **Passo 0: Configurar o Fuso Horário do Sistema**

Antes de qualquer coisa, é essencial garantir que o sistema esteja configurado no fuso horário correto para que os agendamentos no `cron` sejam executados no horário desejado.

```bash
sudo timedatectl set-timezone America/Sao_Paulo
```

- **O que faz**: Ajusta o fuso horário do sistema para o horário de Brasília (BRT/BRST).

---

#### **Passo 1: Criar o Script `kill_o11.sh` para Encerrar o Processo às 4h e 4h03min**

Este script será responsável por encerrar o processo `./o11` de forma agressiva.

##### **1.1. Criar o Script**

```bash
sudo nano /usr/local/bin/kill_o11.sh
```

##### **1.2. Adicionar o Conteúdo ao Script**

Adicione o seguinte conteúdo ao arquivo:

```bash
#!/bin/bash

# Encontra o PID do processo o11
O11_PID=$(pgrep -f '/home/o11_v22b1-DRMStuff')

# Verifica se o processo está em execução
if [ ! -z "$O11_PID" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Encerrando o processo o11 com PID $O11_PID" >> /var/log/kill_o11.log
    kill -9 $O11_PID
else
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Nenhum processo o11 em execução." >> /var/log/kill_o11.log
fi
```

- **Explicação**:
  - **`pgrep -f '/home/o11_v22b1-DRMStuff'`**: Encontra o PID do processo `./o11` com base no comando completo.
  - **`kill -9 $O11_PID`**: Encerra o processo de forma agressiva (SIGKILL).
  - **Logs**: Grava informações sobre a ação no arquivo `kill_o11.log` com timestamp.

##### **1.3. Tornar o Script Executável**

```bash
sudo chmod +x /usr/local/bin/kill_o11.sh
```

---

#### **Passo 2: Criar o Script `kill_o11_on_high_memory.sh` para Monitorar e Encerrar o Processo se o Uso de Memória Exceder 60%**

Este script será executado a cada 5 minutos para monitorar o uso de memória e agir conforme necessário.

##### **2.1. Criar o Script**

```bash
sudo nano /usr/local/bin/kill_o11_on_high_memory.sh
```

##### **2.2. Adicionar o Conteúdo ao Script**

```bash
#!/bin/bash

# Define o limite de uso de memória em porcentagem
MEMORY_LIMIT=60

# Obtém o uso atual de memória em porcentagem
MEMORY_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100.0}')

# Obtém a data e hora atual
CURRENT_TIME=$(date '+%Y-%m-%d %H:%M:%S')

# Verifica se o uso de memória está acima do limite
if (( $(echo "$MEMORY_USAGE > $MEMORY_LIMIT" | bc -l) )); then
    echo "$CURRENT_TIME - Uso de memória ($MEMORY_USAGE%) acima de $MEMORY_LIMIT%. Encerrando o processo o11." >> /var/log/kill_o11_memory.log

    # Encontra o PID do processo o11
    O11_PID=$(pgrep -f '/home/o11_v22b1-DRMStuff')

    # Verifica se o processo está em execução
    if [ ! -z "$O11_PID" ]; then
        echo "$CURRENT_TIME - Encerrando o processo o11 com PID $O11_PID" >> /var/log/kill_o11_memory.log
        kill -9 $O11_PID
    else
        echo "$CURRENT_TIME - Nenhum processo o11 em execução." >> /var/log/kill_o11_memory.log
    fi
else
    echo "$CURRENT_TIME - Uso de memória ($MEMORY_USAGE%) está abaixo de $MEMORY_LIMIT%. Nenhuma ação necessária." >> /var/log/kill_o11_memory.log
fi
```

- **Explicação**:
  - **`MEMORY_LIMIT=60`**: Define o limite de uso de memória em 60%.
  - **`MEMORY_USAGE`**: Calcula o uso atual de memória em porcentagem.
  - **Verificação**: Se o uso de memória exceder 60%, o processo `./o11` é encerrado.
  - **Logs**: Grava informações sobre a ação no arquivo `kill_o11_memory.log` com timestamp.

##### **2.3. Tornar o Script Executável**

```bash
sudo chmod +x /usr/local/bin/kill_o11_on_high_memory.sh
```

---

#### **Passo 3: Criar o Script `clear_logs.sh` para Limpar Logs Antigos**

Este script limpará os logs antigos conforme a retenção definida.

##### **3.1. Criar o Script**

```bash
sudo nano /usr/local/bin/clear_logs.sh
```

##### **3.2. Adicionar o Conteúdo ao Script**

```bash
#!/bin/bash

# Define os caminhos dos arquivos de log
LOG_KILL_O11="/var/log/kill_o11.log"
LOG_KILL_O11_MEMORY="/var/log/kill_o11_memory.log"

# Define a quantidade de dias para manter os logs
DAYS_KILL_O11=7
DAYS_KILL_O11_MEMORY=2

# Limpar logs antigos de kill_o11.log que têm mais de 7 dias
find "$LOG_KILL_O11" -mtime +$DAYS_KILL_O11 -exec rm -f {} \;

# Limpar logs antigos de kill_o11_memory.log que têm mais de 2 dias (48 horas)
find "$LOG_KILL_O11_MEMORY" -mtime +$DAYS_KILL_O11_MEMORY -exec rm -f {} \;

# Opcional: Registrar a limpeza nos logs
echo "Logs limpos em $(date '+%Y-%m-%d %H:%M:%S')" >> /var/log/clear_old_logs.log
```

- **Explicação**:
  - **`find "$LOG_KILL_O11" -mtime +7 -exec rm -f {} \;`**: Remove o arquivo `kill_o11.log` se tiver mais de 7 dias.
  - **`find "$LOG_KILL_O11_MEMORY" -mtime +2 -exec rm -f {} \;`**: Remove o arquivo `kill_o11_memory.log` se tiver mais de 2 dias.
  - **Logs de Limpeza**: Adiciona uma entrada no arquivo `clear_old_logs.log` indicando quando os logs foram limpos.

##### **3.3. Tornar o Script Executável**

```bash
sudo chmod +x /usr/local/bin/clear_logs.sh
```

---

#### **Passo 4: Configurar o `crontab` para Agendar as Tarefas**

Vamos configurar o `crontab` para agendar as execuções dos scripts conforme necessário.

##### **4.1. Editar o `crontab`**

```bash
sudo crontab -e
```

##### **4.2. Adicionar as Entradas ao `crontab`**

Adicione as seguintes linhas ao arquivo do `crontab`:

```plaintext
# Executar kill_o11.sh às 4h da manhã
0 4 * * * /usr/local/bin/kill_o11.sh >> /var/log/kill_o11.log 2>&1

# Executar kill_o11.sh novamente às 4h03min da manhã
3 4 * * * /usr/local/bin/kill_o11.sh >> /var/log/kill_o11.log 2>&1

# Executar kill_o11_on_high_memory.sh a cada 5 minutos
*/5 * * * * /usr/local/bin/kill_o11_on_high_memory.sh >> /var/log/kill_o11_memory.log 2>&1

# Limpar logs diariamente às 00h
0 0 * * * /usr/local/bin/clear_logs.sh
```

- **Explicação das Entradas**:
  - **`0 4 * * *`**: Executa `kill_o11.sh` às 4h da manhã.
  - **`3 4 * * *`**: Executa `kill_o11.sh` novamente às 4h03min da manhã.
  - **`*/5 * * * *`**: Executa `kill_o11_on_high_memory.sh` a cada 5 minutos.
  - **`0 0 * * *`**: Executa `clear_logs.sh` diariamente à meia-noite para limpar logs antigos.

##### **4.3. Salvar e Fechar o Editor**

Dependendo do editor que está utilizando (`nano`, `vim`, etc.), salve e saia:
- **No `nano`**: Pressione `Ctrl + O`, `Enter` para salvar e `Ctrl + X` para sair.
- **No `vim`**: Pressione `Esc`, digite `:wq` e pressione `Enter`.

---

#### **Passo 5: Verificar a Configuração do `crontab`**

Para garantir que todas as entradas foram adicionadas corretamente, execute:

```bash
sudo crontab -l
```

Deve mostrar algo semelhante a:

```plaintext
0 4 * * * /usr/local/bin/kill_o11.sh >> /var/log/kill_o11.log 2>&1
3 4 * * * /usr/local/bin/kill_o11.sh >> /var/log/kill_o11.log 2>&1
*/5 * * * * /usr/local/bin/kill_o11_on_high_memory.sh >> /var/log/kill_o11_memory.log 2>&1
0 0 * * * /usr/local/bin/clear_logs.sh
```

---

#### **Passo 6: Testar os Scripts Manualmente**

Antes de confiar nas execuções automáticas do `cron`, é recomendável testar os scripts manualmente para garantir que funcionam conforme esperado.

##### **6.1. Testar o Script `kill_o11.sh`**

```bash
sudo /usr/local/bin/kill_o11.sh
```

- **Verifique o Arquivo de Log**:

```bash
cat /var/log/kill_o11.log
```

- **O que Esperar**:
  - Se o processo `./o11` estiver em execução, o log deve mostrar que o processo foi encerrado com o PID correspondente.
  - Se o processo não estiver em execução, o log deve indicar que nenhum processo foi encontrado.

##### **6.2. Testar o Script `kill_o11_on_high_memory.sh`**

Primeiro, simule um alto uso de memória para testar o script. Você pode criar um processo que consome muita memória ou ajustar temporariamente o limite no script para um valor mais baixo.

Por exemplo, para testar rapidamente, mude `MEMORY_LIMIT=60` para `MEMORY_LIMIT=1` no script:

```bash
sudo nano /usr/local/bin/kill_o11_on_high_memory.sh
```

Altere:

```bash
MEMORY_LIMIT=60
```

Para:

```bash
MEMORY_LIMIT=1
```

Salve e execute o script:

```bash
sudo /usr/local/bin/kill_o11_on_high_memory.sh
```

- **Verifique o Arquivo de Log**:

```bash
cat /var/log/kill_o11_memory.log
```

- **O que Esperar**:
  - O script deve detectar que o uso de memória excede o limite (1% no exemplo) e tentar encerrar o processo `./o11`.
  - O log deve refletir a ação tomada.

Após testar, não se esqueça de reverter a alteração no script para `MEMORY_LIMIT=60`:

```bash
sudo nano /usr/local/bin/kill_o11_on_high_memory.sh
```

Altere de volta:

```bash
MEMORY_LIMIT=60
```

Salve e feche.

##### **6.3. Testar o Script `clear_logs.sh`**

```bash
sudo /usr/local/bin/clear_logs.sh
```

- **Verifique os Arquivos de Log**:

```bash
cat /var/log/kill_o11.log
cat /var/log/kill_o11_memory.log
cat /var/log/clear_old_logs.log
```

- **O que Esperar**:
  - Os arquivos `kill_o11.log` e `kill_o11_memory.log` devem estar vazios ou conter apenas a mensagem de limpeza (dependendo de como o script está configurado).
  - O `clear_old_logs.log` deve registrar a data e hora da limpeza.

---

#### **Passo 7: Verificar o Serviço `cron`**

Assegure-se de que o serviço `cron` está ativo e funcionando corretamente.

##### **7.1. Verificar o Status do `cron`**

```bash
sudo service cron status
```

- **O que Esperar**:
  - O serviço deve estar ativo (`active (running)`).
  - Se não estiver, inicie o serviço:

    ```bash
    sudo service cron start
    ```

##### **7.2. Reiniciar o Serviço `cron` (se necessário)**

Se você fez alterações no `crontab` e não tem certeza se elas foram aplicadas, reinicie o serviço:

```bash
sudo service cron restart
```

---

### **Resumo Final das Configurações**

1. **Fuso Horário**:
   - Configurado para `America/Sao_Paulo`.

2. **Scripts Criados**:
   - **`kill_o11.sh`**: Encerra o processo `./o11` às 4h e 4h03min.
   - **`kill_o11_on_high_memory.sh`**: Monitora o uso de memória e encerra o processo se exceder 60%, executado a cada 5 minutos.
   - **`clear_logs.sh`**: Limpa logs antigos conforme a retenção definida, executado diariamente à meia-noite.

3. **Agendamentos no `crontab`**:
   - **00h**: Executa `clear_logs.sh` para limpar logs antigos.
   - **04h**: Executa `kill_o11.sh` para encerrar o processo.
   - **04h03min**: Executa `kill_o11.sh` novamente para uma segunda tentativa.
   - **A cada 5 minutos**: Executa `kill_o11_on_high_memory.sh` para monitorar uso de memória.

4. **Manutenção de Logs**:
   - **`kill_o11.log`**: Mantém logs por 7 dias.
   - **`kill_o11_memory.log`**: Mantém logs por 48 horas.
   - **`clear_old_logs.log`**: Registra quando os logs são limpos.

---

### **Considerações Finais**

- **Permissões**: Todos os scripts devem ter permissões executáveis (`chmod +x`).
- **Logs**: Verifique regularmente os arquivos de log para garantir que os scripts estão funcionando conforme esperado.
- **Monitoramento**: Utilize comandos como `ps`, `pgrep`, `free`, e ferramentas como `htop` para monitorar o estado do sistema e dos processos.

### **Comandos Úteis para Monitoramento**

- **Verificar Processos**:

  ```bash
  pgrep -fl '/home/o11_v22b1-DRMStuff'
  ```

- **Verificar Uso de Memória**:

  ```bash
  free -h
  ```

- **Verificar Logs**:

  ```bash
  cat /var/log/kill_o11.log
  cat /var/log/kill_o11_memory.log
  cat /var/log/clear_old_logs.log
  ```

- **Reiniciar Serviço `cron`**:

  ```bash
  sudo service cron restart
  ```

### **Teste Final**

1. **Executar Manualmente Todos os Scripts**:
   - Certifique-se de que cada script funciona conforme esperado ao executá-los manualmente.
   
2. **Verificar Logs Após Execuções**:
   - Confirme que os logs estão sendo registrados corretamente com timestamps.

3. **Monitorar o `cron`**:
   - Assegure-se de que as tarefas agendadas estão sendo executadas nos horários corretos.

---

### **Conclusão**

Seguindo este guia passo a passo, você terá configurado com sucesso a automação para encerrar o processo `./o11` em horários específicos e sob condições de alto uso de memória, além de manter a limpeza dos logs de forma automática. Essa configuração ajudará a manter seu sistema estável e gerenciado de forma eficiente.

Se precisar de mais assistência ou tiver outras dúvidas, não hesite em perguntar!
