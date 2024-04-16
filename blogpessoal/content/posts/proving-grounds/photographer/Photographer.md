+++
title = 'Proving Grounds - Photographer'
date = 2024-04-15T09:00:08-03:00
draft = false
+++

Iniciando a série de postagens sobre desafios/CTFs realizados, decidi por documentar a máquina Photographer (nível Fácil/Easy), disponível gratuitamente no [Proving Grounds Play](https://portal.offsec.com/labs/play). A máquina tem como objetivo o acesso inicial à ela, onde o atacante deve adquirir a chave de usuário, e posteriomente escalonar privilégios (privilege escalation), adquirindo a chave de root.


## Enumeração

Para iniciar o processo de enumeração, realizei um scan de portas via **nmap** junto ao IP disponibilizado.

**Comando utilizado:**\
`nmap -sV -p- --min-rate 10000 192.168.245.76`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem1.png)

Visando facilitar o processo de análise e exploração, adicionei o IP 192.168.245.76 no arquivo /etc/hosts da máquina de atacante, atribuindo o nome **photographer.pg** ao IP.

Na sequência, enumerei o serviço SMB disponível (nas portas 139 e 445).

**Comando utilizado:**\
`smbclient -L \\\\photographer.pg -N`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem2.png)

Na sequência efetuei a conexão no share **sambashare** utilizando credenciais nulas (null credentials).


**Comando utilizado:**\
`smbclient \\\\photographer.pg\\sambashare -N`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem3.png)

Como resultado, foi possível identificar os arquivos **mailsent.txt** e **wordpress.bkp.zip**.

Após baixar o arquivo **mailsent.txt** e analisar seu conteúdo, foi possível identificar um usuário chamado **Daisa** e o e-mail **daisa@photographer.com**:

![](/posts/proving-grounds/photographer/imagem3.1.png)

O arquivo zipado **wordpress.bkp.zip** continha apenas uma instalação default de Wordpress, sem informações sensíveis.


Na sequência efetuei a enumeração do servidor web (webserver) rodando na porta **80** através da ferramenta **dirsearch**.

**Comando utilizado:**\
`dirsearch -w /usr/share/wordlists/dirb/big.txt -x 404 -u http://photographer.pg`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem4.png)

O resultado não trouxe informações interessantes que indiquem um caminho a ser utilizado para acesso inicial.

Ao acessar o webserver na porta **80** foi possível identificar a seguinte tela:

![](/posts/proving-grounds/photographer/imagem5.png)


Dando continuidade ao processo de análise, efetuei a enumeração do webserver rodando na porta **8000**.

**Comando utilizado:**\
`dirsearch -w /usr/share/wordlists/dirb/big.txt -x 404 -u http://photographer.pg:8000`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem6.png)

A ideia de filtrar apenas o código de erro 404 visa permitir a visualização páginas/arquivos com redirecionamentos (301, 302), autenticações (401) ou proibição de acesso (403).

Ao acessar o endereço http://photographer.pg:8000/admin/ o usuário é direcionado a um painel administivo:

![](/posts/proving-grounds/photographer/imagem7.png)
 
## Exploração

Inicialmente testei o processo de autenticação com dados randômicos, monitorando as requisições através do **burpsuite**, com o objetivo de identificar os parâmetros da requisição (request) e resposta (response):

![](/posts/proving-grounds/photographer/imagem8.png)

Como resultado foi possível identificar que a requisição faz uma chamada ao endereço **/api.php?/sessions**, passando os parâmetros **email** e **password**. Caso o usuário não exista, o response retorna em formato json a mensagem **User not found.** No caso da senha estar errada, o retorno de erro é: **Incorrect. Try again or reset your password.**

Como na fase de enumeração foi identificado o e-mail **daisa@photographer.com**, preparei um ataque de força bruta (brute force) através da ferramenta **hydra**, passando como referência o e-mail, os parâmetros e retorno de erro: 

**Comando utilizado:**\
`hydra -l daisa@photographer.com -P /usr/share/wordlists/rockyou.txt 192.168.245.76 -s 8000 http-post-form "/api.php?/sessions:email=^USER^&password=^PASS^:Incorrect. Try again or reset your password." -vv -I`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem9.png)

A ferramenta **hydra** identificou que um conjunto de credenciais permitem a autenticação na plataforma. Ao utilizar as credenciais obtidas, foi possível acessar a plataforma de administração da plataforma **Koken CMS**:

![](/posts/proving-grounds/photographer/imagem10.png)

Após utilizar a plataforma por alguns minutos, procurei por um exploit para o CMS, onde identifiquei a seguinte referência de uma falha que permite upload arbritário de arquivos (autenticado): [Exploit-DB - Arbitrary File Upload (Authenticated)](https://www.exploit-db.com/exploits/48706)

A referência indica que o CMS permite o upload irrestrito de arquivos a partir da manipulação da requisição, onde posteriormente, através da identificação do diretório do webserver onde o arquivo foi inserido, o atacante pode acessar esse arquivo e executar sua webshell/shell reversa.

Na sequência criei uma backdoor em php e dei o nome de **image.php.jpg**:

![](/posts/proving-grounds/photographer/imagem11.png)


Efetuei o upload do arquivo através do botão **Import content** (canto inferior direito) no menu **Library panel**:

![](/posts/proving-grounds/photographer/imagem12.png)


Através da interceptação da requisição via **burpsuite**, editei o conteúdo do parâmetro **name** (linha 18) e o conteúdo do parâmetro **filename** (linha 44), mudando ambos para **image.php**:


![](/posts/proving-grounds/photographer/imagem13.png)

Após essa alteração, selecionei o arquivo enviado ao servidor e no menu **Inspector** (lateral direita), cliquei com o botão direito do mouse na opção **Download File** e copiei o link do arquivo:

![](/posts/proving-grounds/photographer/imagem14.png)

Ao acessar o endereço http://photographer.pg:8000/storage/originals/d2/bf/image.php?cmd=whoami foi possível confirmar a execução de comandos remotamente (RCE) através da webshell enviada ao servidor:

![](/posts/proving-grounds/photographer/imagem15.png)

Na sequência preparei um listener de netcat, onde através da ferramenta online [Reverse Shell Generator](https://www.revshells.com/) gerei uma shell reversa do tipo **nc mkfifo** encodada como **URL encode**, passando o resultado como parâmetro via navegador.

![](/posts/proving-grounds/photographer/imagem16.png)

Esse processo de encoding é necessário para que o comando seja executado corretamente, uma vez que ele possui caracteres especiais que podem ser interpretados de forma não desejada pelo webserver ou pelo navegador.


**Resultado da execução do comando, pegando uma shell como www-data:**

![](/posts/proving-grounds/photographer/imagem17.png)


Dentro do diretório **/home/daisa** foi possível localizar o arquivo **local.txt**, contendo a flag de usuário:

![](/posts/proving-grounds/photographer/imagem18.png)


## Escalonamento de Privilégios

Iniciando a análise de vetores de escalonamento de privilégios (privilege escalation), listei os usuários que possuem terminal ativo:

**Comando utilizado:**\
`cat /etc/passwd | grep "/bin/bash"`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem19.png)

Na sequência fiz a análise de binários que possuem o SUID bit setado. Referência sobre [SUID Privilege Escalation](https://reddyyz.github.io/blog/suid-privilege-escalation)

**Comando utilizado:**\
`find / -type f -perm -04000 -ls 2>/dev/null`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem20.png)

No resultado acima, o binário do PHP em **/usr/bin/php7.2** chamou a atenção por não ser um binário que comumente tem esse tipo de permissão. Através do site [GTFOBins](https://gtfobins.github.io/gtfobins/php/) foi possível identificar como explorar a permissão de SUID para escalonar privilégios.

**Comando utilizado:**\
`/usr/bin/php7.2 -r "pcntl_exec('/bin/sh',['-p']);"`

**Resultado:**\
![](/posts/proving-grounds/photographer/imagem21.png)

Como resultado foi possível escalonar privilégios para o usuário **root**.

Dentro do diretório **/root** foi possível localizar o arquivo **proof.txt** contendo a flag final (root) do desafio:

![](/posts/proving-grounds/photographer/imagem22.png)


## Conclusão

Considerei esse desafio tranquilo de ser feito, possuindo um caminho bem definido tanto para exploração inicial quanto para escalonamento de privilégios. Notei depois do bruteforce com hydra que a senha da conta identificada também estava disponível no arquivo **mailsent.txt**, sendo um fator a se atentar para os próximos desafios, pois as evidências dos desafios fornecem informações importantes que podem economizar tempo/trabalho.

