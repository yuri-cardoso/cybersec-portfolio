# 02 · Laboratório de alvos com rede isolada

**Status:** 🟡 Em andamento
**Data:** 09/11/2026

## Objetivo

Montar o laboratório do capítulo 1 do livro *Testes de Invasão* (Georgia Weidman), adaptado para ferramentas atuais, com as máquinas-alvo numa rede totalmente isolada da rede doméstica e da internet. O primeiro alvo é o Metasploitable 2.

## Por que isolar

Uma máquina propositalmente vulnerável (Metasploitable 2) não pode ter contato com a internet nem com a rede real: ela seria um ponto de entrada trivial para um atacante externo. O isolamento garante que os exercícios de exploração aconteçam num ambiente fechado, onde só o Kali (atacante) e os alvos conversam entre si.

## Topologia

| Máquina | Função | Rede | IP |
|---|---|---|---|
| Kali Linux (host) | Atacante | Host-Only `vboxnet0` | 192.168.56.1 |
| Metasploitable 2 (VM) | Alvo Linux | Host-Only `vboxnet0` | 192.168.56.101 |

Rede: **Host-Only** (`192.168.56.0/24`), sem rota para a internet.

## Montagem

### 1. Metasploitable 2

Download legítimo do projeto (Rapid7) e extração do disco `.vmdk`:

```bash
cd ~/Downloads
unzip metasploitable-linux-2.0.0.zip
```

### 2. Rede isolada no VirtualBox

Gerenciador de Rede → criada a rede Host-Only `vboxnet0` com DHCP ativado. Essa rede conecta host e VMs entre si, sem saída para a internet.

### 3. VM do alvo

- Tipo: Linux / Ubuntu 64-bit
- Memória: 512 MB
- Disco: "usar arquivo de disco virtual existente" → `.vmdk` do Metasploitable
- Adaptador 1: **Host-Only**, `vboxnet0`

## Validação do isolamento

Login no Metasploitable (`msfadmin` / `msfadmin`) e checagem do IP:

```
msfadmin@metasploitable:~$ ip addr
eth0: inet 192.168.56.101/24 brd 192.168.56.255 scope global eth0
```

Teste de alcance do Kali para o alvo (deve responder):

```
$ ping -c2 192.168.56.101
2 packets transmitted, 2 received, 0% packet loss
```

Teste de saída do alvo para a internet (deve falhar, provando o isolamento):

```
msfadmin@metasploitable:~$ ping -c2 8.8.8.8
connect: Network is unreachable
```

✅ **Resultado:** o Kali alcança o alvo, e o alvo não alcança a internet. Ambiente isolado e funcional.

## Snapshots criados

| VM | Snapshot | Descrição |
|---|---|---|
| Metasploitable2 | <!-- preencher --> | Estado limpo, recém-instalado |

> Dica: criar um snapshot "limpo" antes de qualquer ataque permite voltar ao estado inicial em segundos se algum exploit derrubar o alvo.

## Próximos passos

- [ ] Criar snapshot "limpo" do Metasploitable
- [ ] Capítulo 5: coleta de informações (nmap) contra o alvo
- [ ] Adicionar um alvo Windows ao lab

## Lições aprendidas

- **Host-Only isola sem perder conectividade interna:** o atacante alcança os alvos, mas nada escapa para a internet.
- **Validar o isolamento com dois testes** (alcance interno OK + saída externa falhando) é mais confiável do que presumir que a configuração está correta.
- **O Metasploitable já vem como disco pronto (`.vmdk`)**, então basta apontar uma VM nova para ele em vez de instalar do zero.
