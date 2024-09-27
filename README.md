# Vamos criar um guia completo e detalhado para configurar uma nova máquina com DRM O11 para:

1. **Rebootar a máquina diariamente às 04h00 da manhã.**
2. **Monitorar o uso de memória a cada 5min e encerrar o processo `./o11` se exceder 60% da RAM.**
3. **Manter e limpar logs conforme a retenção definida:**
   - **`kill_o11.log`**: Guardar logs por **7 dias** antes de serem limpos.
   - **`kill_o11_memory.log`**: Guardar logs por **48 horas** antes de serem limpos.

### **Visão Geral do Objetivo**

1. **Reboot Automático:**
   - **Horário**: 04h00 diariamente.
2. **Monitoramento de Memória:**
   - **Ação**: Encerrar o processo `./o11` se o uso de memória exceder 60%.
   - **Frequência**: A cada 5 minutos.
3. **Manutenção de Logs:**
   - **`kill_o11.log`**: Guardar logs por **7 dias**.
   - **`kill_o11_memory.log`**: Guardar logs por **48 horas**.
4. **Limpeza Automática de Logs:**
   - **Horário**: 00h00 diariamente.

---

### **Passo a Passo Completo**

#### **Passo 0: Configurar o Fuso Horário do Sistema**

1. Antes de qualquer coisa, é essencial garantir que o sistema esteja no fuso horário correto para que os agendamentos no `cron` sejam executados no horário desejado.
2. Somente faça o Passo 0 se você nunca definiu o fuso da maquina para o Brasil.

```bash
sudo timedatectl set-timezone America/Sao_Paulo
```
```bash
sudo reboot
```

- **O que faz**: Ajusta o fuso horário do sistema para o horário de Brasília (BRT/BRST) e reinicia a maquina para aplicar.

---

#### **Passo 1: Criar o Script `kill_o11_on_high_memory.sh` para Monitorar Uso de Memória**

Este script será responsável por monitorar o uso de memória a cada 5 minutos e encerrar o processo `./o11` se o uso exceder 60%.

##### **1.1. Criar o Script**

```bash
sudo nano /usr/local/bin/kill_o11_on_high_memory.sh
```

##### **1.2. Adicionar o Conteúdo ao Script**

Copie e cole o seguinte conteúdo no editor:

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

- **Explicação:**
  - **`MEMORY_LIMIT=60`**: Define o limite de uso de memória em 60%.
  - **`MEMORY_USAGE`**: Calcula o uso atual de memória em porcentagem.
  - **`CURRENT_TIME`**: Obtém a data e hora atual.
  - **Verificação**: Se o uso de memória exceder 60%, o processo `./o11` é encerrado.
  - **Logs**: Grava informações sobre a ação no arquivo `kill_o11_memory.log` com timestamp.

##### **1.3. Tornar o Script Executável**

```bash
sudo chmod +x /usr/local/bin/kill_o11_on_high_memory.sh
```

---

#### **Passo 2: Criar o Script `clear_logs.sh` para Limpar Logs Antigos**

Este script limpará os logs antigos conforme a retenção definida.

##### **2.1. Criar o Script**

```bash
sudo nano /usr/local/bin/clear_logs.sh
```

##### **2.2. Adicionar o Conteúdo ao Script**

Copie e cole o seguinte conteúdo no editor:

```bash
#!/bin/bash

# Define os caminhos dos arquivos de log
LOG_KILL_O11="/var/log/kill_o11.log"
LOG_KILL_O11_MEMORY="/var/log/kill_o11_memory.log"

# Define a quantidade de dias para manter os logs
DAYS_KILL_O11=7
DAYS_KILL_O11_MEMORY=2

# Limpar logs antigos de kill_o11.log que têm mais de 7 dias
find "$LOG_KILL_O11" -type f -mtime +$DAYS_KILL_O11 -exec truncate -s 0 {} \;

# Limpar logs antigos de kill_o11_memory.log que têm mais de 2 dias (48 horas)
find "$LOG_KILL_O11_MEMORY" -type f -mtime +$DAYS_KILL_O11_MEMORY -exec truncate -s 0 {} \;

# Opcional: Registrar a limpeza nos logs de limpeza
echo "Logs limpos em $(date '+%Y-%m-%d %H:%M:%S')" >> /var/log/clear_old_logs.log
```

- **Explicação:**
  - **`find "$LOG_KILL_O11" -type f -mtime +7 -exec truncate -s 0 {} \;`**: Esvazia o conteúdo do arquivo `kill_o11.log` se tiver mais de 7 dias.
  - **`find "$LOG_KILL_O11_MEMORY" -type f -mtime +2 -exec truncate -s 0 {} \;`**: Esvazia o conteúdo do arquivo `kill_o11_memory.log` se tiver mais de 2 dias.
  - **`truncate -s 0 {}`**: Esvazia o arquivo sem removê-lo, mantendo o arquivo para futuras entradas.
  - **Logs de Limpeza**: Adiciona uma entrada no arquivo `clear_old_logs.log` indicando quando os logs foram limpos.

##### **2.3. Tornar o Script Executável**

```bash
sudo chmod +x /usr/local/bin/clear_logs.sh
```

---

#### **Passo 3: Configurar o Reboot Automático às 04h00**

Vamos agendar o reboot diário às 04h00 da manhã.

##### **3.1. Adicionar a Entrada de Reboot no `crontab`**

Abra o `crontab` para edição:

```bash
sudo crontab -e
```

Adicione a seguinte linha para agendar o reboot às 04h00 diariamente:

```plaintext
0 4 * * * /sbin/shutdown -r now >> /var/log/reboot.log 2>&1
```

- **Explicação:**
  - **`0 4 * * *`**: Executa às 04h00 diariamente.
  - **`/sbin/shutdown -r now`**: Comando para reiniciar imediatamente a máquina.
  - **`>> /var/log/reboot.log 2>&1`**: Redireciona a saída e erros para o arquivo de log `/var/log/reboot.log`.

##### **3.2. Criar o Arquivo de Log para Reboots**

Caso ainda não exista, crie o arquivo de log para registrar os reboots:

```bash
sudo touch /var/log/reboot.log
sudo chmod 644 /var/log/reboot.log
```

---

#### **Passo 4: Configurar o `crontab` para Agendar as Tarefas**

Vamos configurar o `crontab` para agendar as execuções dos scripts conforme necessário.

##### **4.1. Editar o `crontab`**

Abra o `crontab` para edição:

```bash
sudo crontab -e
```

##### **4.2. Adicionar as Entradas ao `crontab`**

Adicione as seguintes linhas ao arquivo do `crontab`:

```plaintext
# Reboot diário às 04h00 da manhã
0 4 * * * /sbin/shutdown -r now >> /var/log/reboot.log 2>&1

# Monitorar uso de memória a cada 5 minutos
*/5 * * * * /usr/local/bin/kill_o11_on_high_memory.sh >> /var/log/kill_o11_memory.log 2>&1

# Limpar logs diariamente às 00h00
0 0 * * * /usr/local/bin/clear_logs.sh >> /var/log/clear_logs.log 2>&1
```

- **Explicação das Entradas:**
  - **Reboot Diário:**
    - **`0 4 * * *`**: Executa às 04h00 diariamente.
    - **`/sbin/shutdown -r now`**: Comando para reiniciar a máquina.
    - **`>> /var/log/reboot.log 2>&1`**: Redireciona a saída e erros para o arquivo de log `/var/log/reboot.log`.
  - **Monitoramento de Memória:**
    - **`*/5 * * * *`**: Executa a cada 5 minutos.
    - **`/usr/local/bin/kill_o11_on_high_memory.sh`**: Caminho para o script de monitoramento.
    - **`>> /var/log/kill_o11_memory.log 2>&1`**: Redireciona a saída e erros para o arquivo de log `kill_o11_memory.log`.
  - **Limpeza de Logs:**
    - **`0 0 * * *`**: Executa diariamente às 00h00.
    - **`/usr/local/bin/clear_logs.sh`**: Caminho para o script de limpeza de logs.
    - **`>> /var/log/clear_logs.log 2>&1`**: Redireciona a saída e erros para o arquivo de log `clear_logs.log`.

##### **4.3. Salvar e Fechar o Editor**

Dependendo do editor que está utilizando (`nano`, `vim`, etc.), salve e saia:

- **No `nano`:** Pressione `Ctrl + O`, `Enter` para salvar e `Ctrl + X` para sair.
- **No `vim`:** Pressione `Esc`, digite `:wq` e pressione `Enter`.

##### **4.4. Verificar a Configuração do `crontab`**

Para garantir que todas as entradas foram adicionadas corretamente, execute:

```bash
sudo crontab -l
```

Deve mostrar algo semelhante a:

```plaintext
0 4 * * * /sbin/shutdown -r now >> /var/log/reboot.log 2>&1
*/5 * * * * /usr/local/bin/kill_o11_on_high_memory.sh >> /var/log/kill_o11_memory.log 2>&1
0 0 * * * /usr/local/bin/clear_logs.sh >> /var/log/clear_logs.log 2>&1
```

---

#### **Passo 5: Testar os Scripts Manualmente**

Antes de confiar nas execuções automáticas do `cron`, é recomendável testar os scripts manualmente para garantir que funcionam conforme esperado.

##### **5.1. Testar o Script `kill_o11_on_high_memory.sh`**

Execute o script manualmente:

```bash
sudo /usr/local/bin/kill_o11_on_high_memory.sh
```

- **Verifique o Arquivo de Log:**

```bash
cat /var/log/kill_o11_memory.log
```

- **O que Esperar:**
  - **Se o uso de memória estiver acima de 60%:**
    - O log deve mostrar que o processo foi encerrado.
  - **Se o uso de memória estiver abaixo de 60%:**
    - O log deve indicar que nenhuma ação foi necessária.

##### **5.2. Testar o Script `clear_logs.sh`**

Execute o script manualmente:

```bash
sudo /usr/local/bin/clear_logs.sh
```

- **Verifique os Arquivos de Log:**

```bash
cat /var/log/kill_o11.log
cat /var/log/kill_o11_memory.log
cat /var/log/clear_old_logs.log
```

- **O que Esperar:**
  - **`kill_o11.log`** e **`kill_o11_memory.log`** devem estar vazios ou conter apenas as últimas entradas dentro do período de retenção.
  - **`clear_old_logs.log`** deve registrar a data e hora da limpeza.

##### **5.3. Testar o Reboot (Opcional)**

**Atenção:** Este passo reiniciará a sua máquina. Execute apenas se estiver seguro e não houver trabalhos críticos em execução.

Execute o reboot manualmente para verificar se está funcionando:

```bash
sudo /sbin/shutdown -r now
```

- **Verifique o Arquivo de Log:**

Após o reboot, verifique o log:

```bash
cat /var/log/reboot.log
```

- **O que Esperar:**
  - O log deve registrar que o comando de reboot foi executado.

**Nota:** Evite reiniciar a máquina frequentemente durante os testes para não interromper serviços essenciais.

---

#### **Passo 6: Verificar o Serviço `cron`**

Assegure-se de que o serviço `cron` está ativo e funcionando corretamente.

##### **6.1. Verificar o Status do `cron`**

```bash
sudo service cron status
```

- **O que Esperar:**
  - O serviço deve estar ativo (`active (running)`).
  - Se não estiver, inicie o serviço:

    ```bash
    sudo service cron start
    ```

##### **6.2. Reiniciar o Serviço `cron` (se necessário)**

Se você fez alterações no `crontab` e não tem certeza se elas foram aplicadas, reinicie o serviço:

```bash
sudo service cron restart
```

---

### **Resumo Final das Configurações**

1. **Fuso Horário:**
   - Configurado para `America/Sao_Paulo`.

2. **Scripts Criados/Atualizados:**
   - **`kill_o11_on_high_memory.sh`**: Monitora o uso de memória e encerra o processo `./o11` se exceder 60%, executado a cada 5 minutos.
   - **`clear_logs.sh`**: Limpa logs antigos conforme a retenção definida, executado diariamente às 00h00.
   - **Reboot Diário**: Agendado para reiniciar a máquina às 04h00 diariamente.

3. **Agendamentos no `crontab`:**
   - **00h00**: Executa `clear_logs.sh` para limpar logs antigos.
   - **04h00**: Executa o comando de reboot para reiniciar a máquina.
   - **A cada 5 minutos**: Executa `kill_o11_on_high_memory.sh` para monitorar uso de memória.

4. **Manutenção de Logs:**
   - **`kill_o11.log`**: Mantém logs por 7 dias.
   - **`kill_o11_memory.log`**: Mantém logs por 48 horas.
   - **`clear_old_logs.log`**: Registra quando os logs são limpos.
   - **`reboot.log`**: Registra os eventos de reboot.

---

### **Considerações Finais**

- **Permissões:**
  - Todos os scripts devem ter permissões executáveis (`chmod +x`).

- **Logs:**
  - Verifique regularmente os arquivos de log para garantir que os scripts estão funcionando conforme esperado.
  - Utilize comandos como `cat`, `tail` e `grep` para analisar os logs.

- **Monitoramento:**
  - Utilize ferramentas como `ps`, `pgrep`, `free`, e `htop` para monitorar o estado do sistema e dos processos.
  - Exemplo para verificar processos:

    ```bash
    pgrep -fl '/home/o11_v22b1-DRMStuff'
    ```

  - Exemplo para verificar uso de memória:

    ```bash
    free -h
    ```

- **Reboot Automático:**
  - Assegure-se de que o reboot diário não interfere em tarefas críticas.
  - Planeje o reboot para um horário de baixa utilização, se possível.

---

### **Comandos Úteis para Monitoramento**

- **Verificar Processos:**

  ```bash
  pgrep -fl '/home/o11_v22b1-DRMStuff'
  ```

- **Verificar Uso de Memória:**

  ```bash
  free -h
  ```

- **Verificar Logs:**

  ```bash
  cat /var/log/kill_o11.log
  cat /var/log/kill_o11_memory.log
  cat /var/log/clear_old_logs.log
  cat /var/log/reboot.log
  ```

- **Reiniciar Serviço `cron`:**

  ```bash
  sudo service cron restart
  ```

---

### **Teste Final**

1. **Executar Manualmente Todos os Scripts:**
   - **Monitoramento de Memória:**

     ```bash
     sudo /usr/local/bin/kill_o11_on_high_memory.sh
     ```

     - **Verifique o log:**

       ```bash
       cat /var/log/kill_o11_memory.log
       ```

   - **Limpeza de Logs:**

     ```bash
     sudo /usr/local/bin/clear_logs.sh
     ```

     - **Verifique os logs:**

       ```bash
       cat /var/log/kill_o11.log
       cat /var/log/kill_o11_memory.log
       cat /var/log/clear_old_logs.log
       ```

   - **Reboot (Opcional):**

     **Atenção:** Este passo reiniciará a sua máquina. Execute apenas se estiver seguro.

     ```bash
     sudo /sbin/shutdown -r now
     ```

     - **Verifique o log:**

       Após o reboot, verifique o log:

       ```bash
       cat /var/log/reboot.log
       ```

2. **Verificar o `crontab`:**
   - Certifique-se de que todas as entradas estão corretas:

     ```bash
     sudo crontab -l
     ```

3. **Monitorar as Execuções Automáticas:**
   - Após os agendamentos, verifique os logs para confirmar que as tarefas estão sendo executadas conforme o esperado.

---

### **Conclusão**

Seguindo este guia passo a passo, você terá configurado com sucesso a automação para:

- **Rebootar a máquina diariamente às 04h00.**
- **Monitorar o uso de memória e encerrar o processo `./o11` se necessário, a cada 5 minutos.**
- **Manter e limpar logs conforme a retenção definida.**

Essa configuração ajudará a manter seu sistema estável e gerenciado de forma eficiente, garantindo que os processos críticos sejam controlados e que os logs sejam mantidos de maneira organizada.

Se precisar de mais assistência ou tiver outras dúvidas, não hesite em me chamar: https://t.me/felipeadmin
