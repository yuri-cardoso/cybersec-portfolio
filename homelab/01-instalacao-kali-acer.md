# 01 · Instalação do Kali Linux em máquina de laboratório dedicada

**Status:** ✅ Concluído
**Data:** 07/11/2026

## Resumo

Instalação do Kali Linux 2026.3 num notebook Acer Aspire VX15 dedicado ao laboratório. O plano inicial era um dual boot no notebook principal, abandonado depois de um diagnóstico de particionamento. Durante a instalação e a configuração surgiram quatro problemas reais, todos documentados abaixo com causa e solução:

1. Redução de partição NTFS bloqueada pela MFT (plano inicial)
2. Falha na etapa "Instalar o sistema básico" causada pela gravação do pendrive
3. Bloqueio de tela do KDE quebrado durante o `full-upgrade`
4. Driver NVIDIA incompatível com o kernel, travando o gerenciador de pacotes

## Ambiente final

| Item | Detalhe |
|---|---|
| Máquina | Acer Aspire VX15 |
| RAM | 16 GB (upgrade) |
| Disco | SSD 512 GB (upgrade), disco inteiro dedicado ao Kali |
| GPU | NVIDIA GeForce GTX 1050 Ti |
| Sistema | Kali Linux 2026.3 (ISO Installer), kernel `7.2.15+kali-amd64` |
| Particionamento | Disco inteiro com LVM criptografado (LUKS) |
| Interface | KDE Plasma |
| Virtualização | VirtualBox 7.2.20 (para as máquinas-alvo) |

---

## Plano inicial: dual boot no notebook principal

A primeira ideia era instalar o Kali em dual boot com o Windows 11 num Alienware m16 R1, para usar a GPU mais forte. Antes de mexer no disco, fiz as checagens de segurança.

### Checagens prévias (PowerShell)

```powershell
# Disco criptografado? Mudanças na BIOS podem exigir a chave de recuperação
manage-bde -status C:
(Get-BitLockerVolume -MountPoint C:).KeyProtector

# Desativar hibernação e inicialização rápida (evita NTFS "travado" para o Linux)
powercfg /h off

# Integridade do sistema de arquivos
chkdsk C: /scan
```

**Resultado:** sem BitLocker, sistema de arquivos sem erros.

### Problema 1: o Windows só permitia reduzir 5 GB

Com **~376 GB livres**, o Gerenciamento de Disco oferecia só **5.336 MB** para redução do volume C:.

**Diagnóstico:** o Windows não reduz uma partição além do último arquivo "não móvel". O log do otimizador de disco aponta qual é:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='Microsoft-Windows-Defrag'} -MaxEvents 3 |
    Select-Object -ExpandProperty Message
```

```
O último arquivo não móvel parece ser: \$Mft::$BITMAP
```

**Causa:** o `$Mft::$BITMAP` faz parte da **Master File Table**, a estrutura central do NTFS. Ele estava perto do fim da partição, e as ferramentas nativas do Windows não conseguem movê-lo com o sistema em execução. As soluções mais citadas para esse erro (desativar arquivo de paginação, despejo de memória e pontos de restauração) **não resolveriam**, porque o bloqueio vinha da própria MFT.

**Solução possível:** redimensionar offline com o particionador do instalador do Kali (`ntfsresize`).

**Decisão:** em vez de arriscar o notebook de uso diário, optei por uma **máquina dedicada ao laboratório**, sem dual boot. Isso elimina o risco ao sistema principal e isola as ferramentas e os dados de teste.

---

## Instalação no Acer

### Validação da ISO

Antes de gravar, conferi a integridade da imagem comparando o hash com o publicado em kali.org:

```powershell
Get-FileHash "C:\Users\<usuario>\Downloads\kali-linux-2026.3-installer-amd64.iso" -Algorithm SHA256
```

```
DB9CBED33EDB45727D16911813C2434FCD1419585007884153BE22A03B30EC2F
```

✅ Idêntico ao SHA256 oficial.

### Problema 2: falha em "Instalar o sistema básico"

O instalador falhava sempre na etapa de instalação do sistema básico.

**Diagnóstico:** como o hash da ISO estava correto, o arquivo baixado foi descartado como causa. Restavam a gravação e o pendrive. O instalador copia os pacotes do pendrive nessa etapa, então qualquer leitura corrompida derruba o processo.

**Solução:** regravei o pendrive no Rufus em **modo Imagem DD** em vez do modo ISO (padrão). O modo DD copia a imagem bit a bit, preservando a estrutura híbrida da ISO do Kali. A instalação passou a funcionar.

### Configuração escolhida

- Particionamento assistido usando o **disco inteiro com LVM criptografado**: o disco todo é protegido por LUKS e a senha é pedida a cada boot, antes do sistema carregar. Numa máquina que guarda hashes, capturas de tráfego e ferramentas de ataque, a criptografia garante que, em caso de perda ou roubo do notebook, nada disso fique acessível.
- Interface **KDE Plasma** (comparada com Xfce e GNOME antes da escolha)

---

## Pós-instalação

### Problema 3: tela travada durante o `full-upgrade`

```bash
sudo apt update && sudo apt full-upgrade -y
```

No meio da atualização, a tela bloqueou sozinha e o bloqueio do KDE quebrou, exibindo a mensagem pedindo `loginctl unlock-session` a partir de um terminal virtual.

**Causa:** o próprio pacote do bloqueio de tela (`kscreenlocker`) foi atualizado enquanto estava em execução.

**Recuperação, sem reiniciar no meio da atualização:**

1. Acesso a um terminal virtual com `Ctrl+Alt+F3` (o F2 não respondeu)
2. Confirmação de que a atualização seguia rodando:
   ```bash
   pgrep -a apt
   # 2222 apt full-upgrade -y
   ```
3. A sessão gráfica ficou com tela preta mesmo após o desbloqueio, então acompanhei o fim da atualização pelo log:
   ```bash
   tail -f /var/log/apt/term.log
   # ... Log ended
   ```
4. Garantia de pacotes consistentes e reinício:
   ```bash
   sudo dpkg --configure -a
   sudo reboot
   ```

**Por que não reiniciar direto:** interromper o `dpkg` no meio pode deixar pacotes meio configurados e o sistema sem boot.

### Problema 4: driver NVIDIA travando o apt

```bash
sudo apt install -y linux-headers-amd64 nvidia-driver nvidia-cuda-toolkit virtualbox
```

- O `nvidia-cuda-toolkit` estava indisponível no repositório (confirmado que os componentes `contrib` e `non-free` estavam habilitados em `/etc/apt/sources.list.d/*.sources`, novo formato deb822 do Kali).
- Sem o CUDA, a instalação do driver falhou:

```
dpkg: problemas de dependência impedem a configuração de nvidia-driver:
 nvidia-driver depende de nvidia-kernel-dkms (= 550.163.01-5.1)
 O pacote nvidia-kernel-dkms ainda não está configurado.
Erros foram encontrados durante o processamento de:
 nvidia-kernel-dkms
 nvidia-driver
```

**Efeito colateral:** com pacotes desconfigurados, o apt passou a falhar em **qualquer** instalação, inclusive do Bluetooth.

**Diagnóstico:**

```bash
dkms status
# virtualbox/7.2.20, 7.2.15+kali-amd64, x86_64: installed
uname -r
# 7.2.15+kali-amd64
```

O módulo do VirtualBox compilou normalmente para o kernel 7.2.15, então headers e ferramentas de compilação estavam corretos. O problema era específico do **driver NVIDIA 550**, mais antigo que o kernel e incompatível com ele.

**Solução:** remover o driver para destravar o gerenciador de pacotes. O sistema segue com o driver livre `nouveau`.

```bash
apt purge -y 'nvidia-*' 'libnvidia-*'
apt autoremove -y
dpkg --configure -a
```

**Pendência:** reinstalar o driver NVIDIA quando o repositório oferecer uma versão compatível com o kernel atual.

### Bluetooth

O Kali vem com o Bluetooth desativado por padrão, como medida de redução de superfície de ataque. Ativação:

```bash
apt install -y bluez bluedevil
systemctl enable --now bluetooth
rfkill unblock bluetooth
bluetoothctl show
```

✅ Funcionando.

---

## Checklist final

- [x] ISO validada por SHA256
- [x] Kali instalado em disco dedicado com LVM criptografado (LUKS)
- [x] Sistema atualizado
- [x] VirtualBox instalado e módulo compilado para o kernel atual
- [x] Bluetooth ativado
- [ ] Driver NVIDIA (aguardando versão compatível com o kernel 7.2)

## Lições aprendidas

- **Espaço livre não é espaço redutível.** A posição física dos arquivos de sistema define até onde uma partição pode encolher.
- **Diagnosticar antes de aplicar receita pronta.** O log do `Defrag` e o `dkms status` mostraram a causa exata, evitando mudanças desnecessárias.
- **Validar a mídia por hash** separa rapidamente "arquivo corrompido" de "gravação ruim".
- **Nunca reiniciar com o `dpkg` rodando.** Terminais virtuais (`Ctrl+Alt+F2..F6`) permitem acompanhar e recuperar o sistema mesmo com a interface gráfica travada.
- **Um pacote quebrado trava o apt inteiro.** Resolver o pacote com problema é pré-requisito para qualquer outra instalação.
- **Separar a máquina de laboratório da máquina de uso diário** reduz riscos e simplifica a manutenção.
