# 📓 Caderno de comandos

Anotações pessoais de comandos que vou aprendendo no Kali/Linux. Cada um com o que faz e um exemplo.

---

## Hardware e sistema

| Comando | O que faz |
|---|---|
| `lspci` | Lista os dispositivos conectados ao barramento PCI (placa de vídeo, rede, etc.) |
| `nvidia-smi` | Mostra a placa de vídeo NVIDIA e o uso dela (só funciona com o driver instalado) |
| `uname -r` | Mostra a versão do kernel do Linux em uso |

```bash
lspci | grep -i nvidia   # filtra só a placa NVIDIA
uname -r                 # ex.: 7.2.15+kali-amd64
```

> Dica: o `grep -i` filtra a saída por uma palavra (o `-i` ignora maiúscula/minúscula). Funciona com qualquer comando que gere muita informação.

---

## Sistema de arquivos e dispositivos

No Linux "tudo é arquivo", inclusive o hardware. Os dispositivos aparecem como arquivos dentro de `/dev/`.

**Pastas especiais:**
| Pasta | O que é |
|---|---|
| `/dev/` | Os dispositivos do computador vistos como arquivos (`dev` = device) |
| `/proc/` | Falso sistema de arquivos com dados dos processos e do kernel (criado na memória, não existe no disco) |
| `/sys/` | Falso sistema de arquivos com dados dos dispositivos e do hardware (mais novo e organizado que o `/proc`) |

**Como os discos são nomeados:**
- `sd` vem de *SCSI disk*, mas hoje vale pra qualquer disco comum (SATA, HD, SSD, pendrive). **Não é "solid disk".**
- A letra é a ordem: `sda` (1º disco), `sdb` (2º disco)...
- O número é a partição: `sda1` (1ª partição do 1º disco), `sda2`...
- SSD **NVMe** não usa `sd`, aparece como `nvme0n1`.

**Outros dispositivos em `/dev/`:**
| Nome | O que é |
|---|---|
| `/dev/sda`, `/dev/nvme0n1` | Discos |
| `/dev/ttyS0` | Porta **serial** (`tty` = teletype, origem do terminal; `S` = serial). Equivale ao "COM1" do Windows |
| `/dev/ttyUSB0` | Adaptador **USB-serial** (usado pra acessar roteadores, câmeras e IoT em hardware hacking) |
| `/dev/input/event0` | Teclado |
| `/dev/input/mouse0` | Mouse |

```bash
lsblk                # mostra discos e partições em árvore
cat /proc/cpuinfo    # dados da CPU
cat /proc/meminfo    # dados da memória RAM
```

### Dispositivos de bloco vs caractere

- **Bloco (`b`)** = como um livro: tamanho definido, você acessa qualquer ponto (acesso aleatório). São os discos: `/dev/sda`, `/dev/nvme0n1`.
- **Caractere (`c`)** = como uma ligação: um fluxo de dados em sequência, sem pular nem voltar. São teclado, mouse, portas seriais, som.

O primeiro caractere do `ls -l` diz o tipo:

| Letra | Tipo |
|---|---|
| `-` | Arquivo comum |
| `d` | Diretório |
| `l` | Atalho (link simbólico) |
| `b` | Dispositivo de bloco |
| `c` | Dispositivo de caractere |
| `p` | Pipe |
| `s` | Socket |

```bash
ls -l /dev/sda       # começa com b
ls -l /dev/ttyS0     # começa com c
```

**Dispositivos de caractere "mágicos" (não são hardware, são ferramentas):**
| Arquivo | O que faz |
|---|---|
| `/dev/null` | Buraco negro: tudo que você joga nele some. Serve pra descartar saída indesejada |
| `/dev/zero` | Fonte infinita de zeros |
| `/dev/urandom` | Fonte infinita de dados aleatórios (senhas, chaves) |

```bash
# dd copia de um "input file" (if) pra um "output file" (of)
# CUIDADO: apaga tudo. Nunca aponte pro disco errado.
sudo dd if=/dev/urandom of=/dev/sdX   # apaga um disco escrevendo dados aleatórios por cima
```

### Hierarquia de pastas do Linux (FHS)

O Linux organiza as pastas de forma padrão (FHS). As 🔥 são as mais relevantes pra segurança.

| Pasta | O que guarda |
|---|---|
| `/bin/` `/sbin/` | Programas básicos e de sistema (onde ficam os comandos) |
| `/boot/` | Kernel e arquivos de inicialização |
| `/dev/` | Arquivos de dispositivo |
| 🔥 `/etc/` | Arquivos de configuração. Inclui `/etc/passwd` (usuários) e `/etc/shadow` (hashes de senha) |
| 🔥 `/home/` | Arquivos pessoais dos usuários (senhas salvas, `~/.ssh/`, `~/.bash_history`) |
| `/lib/` | Bibliotecas básicas |
| `/media/` | Montagem de dispositivos removíveis (pendrive, CD) |
| `/mnt/` | Ponto de montagem temporário |
| `/opt/` | Programas de terceiros |
| 🔥 `/root/` | Arquivos pessoais do administrador (root) |
| `/run/` | Dados temporários que somem no boot |
| `/srv/` | Dados de servidores hospedados na máquina |
| 🔥 `/tmp/` | Arquivos temporários, qualquer um escreve, esvazia no boot (atacante larga payload aqui) |
| `/usr/` | Programas (subdividido em `bin`, `sbin`, `lib`); `/usr/local/` é pra instalar à mão |
| 🔥 `/var/` | Dados variáveis. Inclui `/var/log/` (logs, ouro pro Blue Team; atacante tenta apagar) |
| `/proc/` `/sys/` | Falsos sistemas de arquivos do kernel (não são do FHS) |

> Olhar de segurança: pergunte sempre "o que um atacante procuraria nessa pasta? o que um defensor protegeria?".

---

### Dotfiles e o padrão XDG

**Dotfiles** são arquivos de configuração na pasta pessoal que começam com ponto (ex.: `.bashrc`). Por isso ficam **ocultos**: só aparecem com `ls -a`.

O padrão **XDG** (do FreeDesktop.org, projeto que cria padrões comuns entre KDE, GNOME, Xfce) organiza assim:
| Pasta | Guarda |
|---|---|
| `~/.config/` | Configurações dos programas |
| `~/.cache/` | Cache (pode apagar sem medo) |
| `~/.local/` | Dados dos aplicativos |

```bash
ls -a ~          # mostra os arquivos ocultos
ls ~/.config     # configurações dos programas
```

🔥 **Dotfiles sensíveis** (alvo de atacante e cuidado ao publicar no GitHub):
| Arquivo | Risco |
|---|---|
| `~/.bash_history` / `~/.zsh_history` | Histórico de comandos, às vezes com senha digitada |
| `~/.ssh/` | Chaves SSH (a privada **nunca** pode vazar) |
| `~/.config/`, `~/.git-credentials` | Tokens e logins de programas |

---

### Sistemas de arquivos (ext2, ext3, ext4)

A diferença entre eles gira em torno do **journaling (diário)**: antes de mexer no disco, o sistema anota o que vai fazer. Se faltar energia no meio, ao religar ele lê o diário e conserta rápido, em vez de varrer o disco inteiro.

| | ext2 | ext3 | ext4 |
|---|---|---|---|
| Journaling | ❌ Não | ✅ Sim | ✅ Sim (melhor) |
| Uso hoje | Pendrive/cartão SD (escreve menos, desgasta menos a memória) | Antigo | **Padrão atual** |

- **ext4** é o que usar em quase tudo (é o padrão do Kali).
- **ext2** só faz sentido em pendrive/cartão SD.

**Formatar uma partição** (`mkfs` = MaKe FileSystem). Operação destrutiva, apaga tudo:
```bash
sudo mkfs.ext4 /dev/sdb1     # formata sdb1 como ext4 (confira o disco com lsblk antes!)
```

Outros sistemas de arquivos úteis:
| Nome | Onde aparece |
|---|---|
| `vfat` / `FAT32` | Pendrives, compatível com Windows |
| `ntfs` | Discos do Windows |
| `nfs` | Sistema de arquivos **de rede**: os dados ficam num servidor, não no disco local |

---

## Informações do sistema (reconhecimento)

Primeiros comandos pra rodar no seu Kali ou num alvo invadido, pra saber onde está pisando.

### Memória e disco
```bash
free -h      # memória RAM legível
df -h        # espaço em disco de cada partição
```
> O `-h` (human readable) deixa os tamanhos legíveis (GB/MB) em vez de blocos de 1KB.

### Identidade
```bash
id           # usuário, ID e grupos
whoami       # só o nome
groups       # só os grupos
```
🔥 Os **grupos** do `id` podem ser caminho pra root: `sudo`, `docker`, `lxd`, `disk`. Num alvo, `id` é dos primeiros comandos.

### Kernel e sistema
```bash
uname -a     # kernel, versão, arquitetura, hostname
uname -r     # só a versão do kernel
hostnamectl  # resumo do sistema
```
🔥 A versão do kernel (`uname -r`) revela possíveis exploits: kernel velho = procurar "kernel X exploit" pra escalar privilégio.

### Logs
```bash
dmesg                      # mensagens do kernel (USB, hardware, erros)
sudo dmesg | grep -i usb   # filtra eventos de USB
journalctl                 # todos os logs (mais antigo primeiro)
journalctl -r              # mais recentes primeiro
journalctl -f              # acompanha ao vivo
journalctl -u ssh          # só os logs do SSH
```
> Blue Team: `journalctl -u ssh` mostra tentativas de login (força bruta aparece aqui).

### Hardware
```bash
lspci        # dispositivos PCI (vídeo, rede)
lsusb        # dispositivos USB
lsblk        # discos e partições
sudo lshw    # descrição completa de tudo
```

---

## Rede

| Comando | O que faz |
|---|---|
| `ip addr` | Mostra as interfaces de rede e os IPs da máquina |
| `ping` | Testa se consegue alcançar outro host (e se tem conexão) |
| `rfkill` | Lista e controla os rádios sem fio (wi-fi e bluetooth) |

```bash
ip addr                        # ver meu IP
ping -c2 192.168.56.101        # manda 2 pings pro alvo
rfkill list                    # ver o que está bloqueado
rfkill unblock bluetooth       # liga o bluetooth
```

> No teste de isolamento do lab, o `ping` pro alvo respondeu e o `ping 8.8.8.8` falhou, provando que o alvo não tem internet.

---

## Variáveis de ambiente e PATH

Uma **variável** é uma caixa com nome e conteúdo. O **`$`** na frente significa "me dê o valor de".

- `PATH` (sem `$`) = a palavra PATH
- `$PATH` (com `$`) = o conteúdo guardado na variável

```bash
echo PATH      # imprime a palavra: PATH
echo $PATH     # imprime o conteúdo: /usr/local/sbin:/usr/bin:/bin:...
```

O `echo` só imprime o que recebe. Quem troca `$PATH` pelo valor é o **shell**, antes do echo rodar.

**O que o PATH faz:** é a lista de pastas (separadas por `:`) onde o shell procura os comandos. Ao digitar `ls`, ele procura nessas pastas na ordem até achar o executável, por isso você não precisa digitar `/bin/ls`.

**Variáveis úteis:**
```bash
echo $HOME     # minha pasta pessoal (/home/usuario)
echo $USER     # meu nome de usuário
echo $SHELL    # qual shell eu uso
```

**Descobrir onde está um comando:**
| Comando | O que faz |
|---|---|
| `which ls` | Mostra o caminho do programa (`/bin/ls`) |
| `type cd` | Diz o tipo: programa em arquivo ou comando interno do shell (builtin) |

> `which` só acha programas em arquivo. `type` é mais completo: avisa quando o comando é embutido no shell (como `cd` e `pwd`), que não têm arquivo próprio.

### Aspas duplas vs simples

| Aspas | Comportamento |
|---|---|
| `"duplas"` | O shell **troca as variáveis** pelo valor |
| `'simples'` | Tudo **literal**, o shell não mexe em nada |

```bash
echo "Minha pasta é $HOME"    # Minha pasta é /home/usuario
echo 'Minha pasta é $HOME'    # Minha pasta é $HOME
```

> Teclado US internacional (teclas mortas): pra digitar aspas no terminal, aperta a tecla e depois **espaço**. Sem isso sai `¨` ou acento, e o comando quebra.

### Imprimir na tela: echo e printf

`echo` é o "imprime na tela" do shell, equivalente ao `print()` do Python, `printf()` do C, `std::cout` do C++, `console.log()` do JS.

O shell tem os dois:
```bash
echo "Olá"                           # simples, pula linha sozinho
printf "Olá\n"                       # controla a quebra com \n
printf "%s tem %d anos\n" "Ana" 30   # com formato, igual ao C (%s texto, %d número)
```

- `echo`: simples e pula linha automática. Bom pra coisa rápida.
- `printf`: usa formato e não pula linha sozinho. **Preferir em scripts sérios**, porque se comporta igual em qualquer sistema (o `echo` varia).

---

## Serviços (ligar e parar programas de fundo)

| Comando | O que faz |
|---|---|
| `systemctl start <serviço>` | Liga um serviço agora |
| `systemctl stop <serviço>` | Para um serviço |
| `systemctl enable --now <serviço>` | Liga agora e deixa ligado em todo boot |

```bash
systemctl start nessusd        # liga o Nessus só quando for usar
systemctl enable --now bluetooth
```

> `start` liga só agora. `enable` deixa ligado pra sempre (gasta recurso à toa se for algo pesado, tipo o Nessus).

---

## Pacotes (instalar e consertar programas)

| Comando | O que faz |
|---|---|
| `sudo apt update` | Atualiza a lista de programas disponíveis |
| `sudo apt full-upgrade -y` | Atualiza tudo que está instalado |
| `sudo apt install <pacote>` | Instala um programa do repositório |
| `sudo apt install ./arquivo.deb` | Instala um `.deb` baixado (o `./` é obrigatório) |
| `sudo dpkg --configure -a` | Conserta instalações que ficaram pela metade |

> Pra usar poderes de administrador, é só `sudo` na frente do comando. O `sudo su` vira root de vez, mas raramente é necessário.

---

## Processos e logs

| Comando | O que faz |
|---|---|
| `pgrep -a <nome>` | Mostra se um programa está rodando (e o número dele, o PID) |
| `tail -f <arquivo>` | Acompanha um arquivo de log ao vivo (Ctrl+C pra sair) |

```bash
pgrep -a apt                        # ver se uma atualização ainda roda
tail -f /var/log/apt/term.log       # acompanhar a atualização
```

### Primeiro plano e segundo plano (jobs)

| Comando / Tecla | O que faz |
|---|---|
| `comando &` | Roda em **segundo plano** e libera o terminal na hora |
| `jobs` | Lista os processos em segundo plano (com o número do job) |
| `fg %1` | Traz o job 1 pro **primeiro plano** |
| **Ctrl+Z** | **Pausa** (congela) o processo em primeiro plano e devolve o terminal |
| `bg %1` | Continua o job 1 pausado, mas em segundo plano |
| `kill %1` | Encerra o job 1 |
| `nohup comando &` | Mantém o processo vivo mesmo fechando o terminal |

```bash
sleep 300 &      # [1] 12345  → job 1, PID 12345
jobs             # lista os jobs
fg %1            # traz pra frente
# Ctrl+Z         # pausa
bg %1            # continua em segundo plano
kill %1          # encerra
```

> Uso prático: esqueceu o `&` num scan demorado? **Ctrl+Z** e depois `bg`. Cuidado: fechar a janela do terminal mata os jobs dela (use outra aba ou `nohup`).

---

## Buscar arquivos: find

`find` procura os arquivos em si (por nome, tipo, permissão, data). Diferente do `grep`, que procura texto dentro deles.

```
find   /   -perm -4000   -type f   2>/dev/null
 (1)  (2)      (3)          (4)         (5)
```
1. `find`: o comando de busca
2. `/`: onde procurar (a raiz = sistema inteiro). Podia ser `/home`, etc.
3. `-perm -4000`: filtro de permissão. `4000` é o setuid; o `-` na frente = "que tenha pelo menos esse bit"
4. `-type f`: só arquivos (`f`). `d` seria só pastas
5. `2>/dev/null`: esconde os erros (ver redirecionamento abaixo)

Outros exemplos:
```bash
find /home -name "*.txt"       # arquivos .txt dentro de /home
find / -type d -name config    # pastas chamadas config
find . -size +100M             # arquivos maiores que 100 MB na pasta atual
```

### Redirecionamento e canais

Todo programa tem três canais:
| Nº | Canal | O que é |
|---|---|---|
| `0` | stdin | entrada (o que você digita) |
| `1` | stdout | saída normal (o resultado) |
| `2` | stderr | saída de erro |

```bash
comando > saida.txt      # manda a saída normal pra um arquivo (sobrescreve)
comando >> saida.txt     # mesma coisa, mas adiciona no fim
comando 2>/dev/null      # joga os erros no lixo (buraco negro)
comando > tudo.txt 2>&1  # junta saída e erros no mesmo arquivo
```

> `2>/dev/null` = "pega o canal de erro (2) e manda pro /dev/null". Por isso o `find` fica limpo, sem os "Permission denied".

---

## Buscar texto: grep

`grep` procura linhas que contêm um texto (ou expressão regular) dentro de arquivos.

| Opção | O que faz |
|---|---|
| `-i` | Ignora maiúscula/minúscula |
| `-n` | Mostra o número da linha |
| `-v` | Inverte: mostra o que NÃO combina |
| `-c` | Conta as linhas que combinam |
| `-r` | Busca recursiva em todos os arquivos de uma pasta |
| `-l` | Mostra só o nome dos arquivos que contêm |
| `-E` | Expressão estendida (ex.: `"a\|b"` vira `"a|b"`) |

```bash
grep -i "maçã" frutas.txt             # busca simples ignorando maiúscula
grep -E "uva|banana" frutas.txt       # uma OU outra
grep -rl "texto" ~/.config/           # quais arquivos da pasta têm "texto"
lspci | grep -i nvidia                # filtrar a saída de outro comando
grep "open" ~/lab/metasploitable/nmap/inicial.txt   # só as portas abertas do nmap
```

---

## Arquivos

| Comando | O que faz |
|---|---|
| `cat <arquivo>` | Mostra o conteúdo de um arquivo de texto |
| `echo "texto" > arquivo.txt` | Escreve um texto num arquivo (sobrescreve) |
| `unzip arquivo.zip` | Descompacta um `.zip` |
| `unzip -o arquivo.zip -d pasta/` | Descompacta sobrescrevendo, numa pasta específica |

---

## Permissões (chmod, chown, setuid)

Uma das bases de segurança: escalação de privilégio quase sempre passa por permissões.

### Ler as permissões (`ls -l`)

Uma linha do `ls -l` tem várias colunas, e o `rwx` é só a primeira:
```
-rw-rw-r--  1  tazladarr  tazladarr  31  nov 9 20:40  frutas.txt
```
| Coluna | O que é |
|---|---|
| `-rw-rw-r--` | As permissões (o rwx fica aqui) |
| `1` | Número de links |
| `tazladarr` (1º) | O dono |
| `tazladarr` (2º) | O grupo |
| `31` | Tamanho em bytes |
| `nov 9 20:40` | Data de modificação |
| `frutas.txt` | Nome |

> O dono e o grupo aparecem com o mesmo nome porque, ao criar um usuário, o Linux cria um grupo com o mesmo nome dele.

Lendo só a coluna das permissões, em blocos de 3:
```
-rw-rw-r--   →   -    rw-    rw-    r--
                tipo  dono  grupo  outros
```
- `-` arquivo / `d` diretório
- `r`=ler (4), `w`=escrever (2), `x`=executar (1)
- Três blocos: dono (`u`), grupo (`g`), outros (`o`)

### chmod octal (r=4, w=2, x=1, soma por categoria)

```bash
chmod 755 script.sh    # rwx / r-x / r-x  → programas e pastas
chmod 644 dados.txt    # rw- / r-- / r--  → arquivos de dados
chmod 600 senha.txt    # rw- / --- / ---  → só o dono lê
chmod 700 ~/.ssh       # só o dono  → padrão pra chaves SSH
```

### chmod simbólico (mexe só num direito)

```bash
chmod +x script.sh     # adiciona execução pra todos
chmod u+x script.sh    # só pro dono
chmod o-r dados.txt    # remove leitura dos outros
```
`u`=dono `g`=grupo `o`=outros `a`=todos · `+` adiciona `-` remove `=` define

### chown e chgrp (dono e grupo)

```bash
sudo chown usuario arquivo          # muda o dono
sudo chown usuario:grupo arquivo    # muda dono e grupo juntos
sudo chown -R usuario pasta/        # recursivo (pasta inteira)
```

### 🔥 setuid, setgid, sticky

- **setuid (`s`)**: o programa roda com o poder do **dono**, não de quem executa. Ex.: `passwd` (deixa você mudar a senha sem ser root).
- **sticky (`t`)**: no `/tmp`, só o dono do arquivo apaga o próprio arquivo.

```bash
chmod 4755 programa    # liga o setuid (4=setuid, 2=setgid, 1=sticky)

# Enumeração de privilégio: achar programas setuid (clássico de pentest)
find / -perm -4000 -type f 2>/dev/null
```

**Como ler a lista num alvo:**
- Setuid **esperado** (normal em qualquer sistema): `sudo`, `su`, `passwd`, `mount`, `pkexec`, `chsh`... deixa quieto.
- Setuid **suspeito** (red flag, dá pra abusar): programas comuns que NÃO deveriam ter setuid, como `find`, `vim`, `nano`, `python`, `bash`, `cp`, `less`, `nmap`.
- A lista do seu próprio Kali é a **linha de base** do que é normal. No alvo, você procura o que foge disso.

> **GTFOBins** (gtfobins.github.io): a bíblia disso. Joga o nome do binário lá e ele diz se e como abusar dele pra escalar privilégio.

### umask (permissão padrão de arquivo novo)

```bash
umask          # ex.: 0022 → remove escrita de grupo e outros nos arquivos novos
```

---

## Git (enviar o portfólio pro GitHub)

| Comando | O que faz |
|---|---|
| `git add .` | Marca todos os arquivos alterados pra enviar |
| `git commit -m "mensagem"` | Salva um ponto com uma descrição do que mudou |
| `git push` | Envia pro GitHub |
| `git status` | Mostra o que mudou e o que falta enviar |

```bash
git add .
git commit -m "Lab 02: rede isolada"
git push
```

> Fluxo do dia a dia: `add` → `commit` → `push`. O `git status` antes ajuda a conferir o que vai subir.
