# TIL: Recuperação de Acesso Sudo em Instância OCI (Oracle Cloud)

**Data:** Outubro de 2026
**Tecnologias:** Oracle Cloud Infrastructure (OCI), Ubuntu Linux (ARM64), Cloud Shell, GRUB

## 1. Sintomas / O Problema
* Perda da chave pública (`public.key`) original gerada durante a criação da instância.
* Acesso remoto mantido apenas por meio de um usuário secundário (`deploy`).
* O usuário `deploy` **não possuía privilégios de administrador** (`sudo`), resultando em "permissão negada" para executar qualquer comando de infraestrutura, leitura de logs ou instalação de pacotes.
* A máquina tornou-se administrativamente inacessível, impedindo a redefinição de senhas.

## 2. Causa Raiz
* **Gestão de Acessos:** O usuário `deploy` foi criado no sistema, mas não foi adicionado ao grupo `sudo` antes da perda da chave principal (usuário `ubuntu`).
* **Bloqueio de Recuperação Local:** A tentativa de interceptar o boot (GRUB) pelo Cloud Shell (Console Serial da OCI) para acessar o shell de emergência (Initramfs / `/bin/bash`) falhou. O motivo é que as imagens de Ubuntu fornecidas pela Oracle Cloud vêm configuradas de fábrica com o parâmetro `GRUB_TIMEOUT=0`. Isso faz com que o Linux carregue instantaneamente, não dando tempo hábil para interceptar o teclado na UEFI.

## 3. Solução Aplicada: Resgate por Volume Secundário (Detach/Attach)
Como o acesso via GRUB não foi possível, a técnica definitiva foi manipular o disco corrompido através de uma máquina virtual saudável, como se fosse um pendrive.

### Passo a Passo Executado:
1. **Isolamento do Disco Original:**
   * Desligar a instância principal (ex: `atos-logos-prod-01`).
   * No painel da OCI, ir até *Boot Volumes* e realizar o **Detach** do disco da máquina.
2. **Criação da Máquina de Resgate:**
   * Subir uma nova instância temporária básica e gratuita (AMD) na mesma rede (VCN).
3. **Anexação do Disco (Attach):**
   * Na máquina temporária, acessar a aba *Attached Block Volumes* e anexar o disco da máquina original como um disco secundário (tipo *Paravirtualized*).
4. **Montagem do Disco:**
   * Acessar a máquina temporária via SSH e listar os discos disponíveis:
     ```bash
     lsblk
     ```
   * Identificar o disco anexado (geralmente `/dev/sdb1` ou `/dev/sda16`) e montá-lo:
     ```bash
     sudo mount /dev/sdb1 /mnt
     ```
5. **Correção de Permissões (Modificação Direta):**
   * Editar o arquivo de grupos do disco original para conceder poder administrativo ao usuário:
     ```bash
     sudo nano /mnt/etc/group
     ```
   * Encontrar a linha do grupo `sudo` e adicionar o usuário `deploy` (ex: de `sudo:x:27:ubuntu` para `sudo:x:27:ubuntu,deploy`).
6. **Retorno à Máquina Original:**
   * Desmontar o volume com segurança:
     ```bash
     sudo umount /mnt
     ```
   * No painel da OCI, desanexar (Detach) o disco da máquina temporária e anexá-lo novamente à máquina original como **Boot Volume**.
7. **Validação e Limpeza:**
   * Iniciar a instância principal. Logar com o usuário `deploy` e executar o comando `sudo su -` com sucesso.
   * Excluir (Terminate) a máquina temporária certificando-se de marcar a opção **"Permanently delete the attached boot volume"** para manter a conta livre de cobranças.

## 4. Prevenção e Aprendizados
* **Validar acessos antes de sair:** Nunca encerrar a sessão do usuário administrador (root/ubuntu) sem antes abrir um segundo terminal, conectar com o novo usuário (`deploy`) e confirmar que ele tem permissões testando um comando como `sudo -l`.
* **Desocultar o menu do GRUB:** Assim que criar uma instância Ubuntu na OCI, alterar imediatamente as configurações de boot editando `/etc/default/grub`. Mudar `GRUB_TIMEOUT_STYLE` para `menu` e definir `GRUB_TIMEOUT=5`, finalizando com o comando `sudo update-grub`. Isso garantirá uma janela de 5 segundos em todos os boots para acesso emergencial.