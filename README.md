# Recuperação e remoção completa do ViPER4Android-RE no KernelSU

## Objetivo

Este guia registra o procedimento usado no Galaxy S23 Ultra para remover o módulo `ViPER4Android-RE` quando ele fica parcialmente aplicado: o KernelSU pode indicar que o módulo foi desativado/removido, mas ainda existem montagens (`mounts`) do módulo no namespace do sistema. Isso pode deixar o áudio inconsistente ou impedir uma reinstalação limpa.

**Importante:** este procedimento é para o caso específico documentado aqui. Ele desmonta apenas entradas cuja origem no `mountinfo` pertence ao diretório do ViPER. Não remova diretórios de outros módulos e não execute `rm -rf` no módulo enquanto ainda houver montagens dele.

## O que aconteceu neste caso

- Módulo: `ViPER4Android-RE`
- Diretório: `/data/adb/modules/ViPER4Android-RE`
- Root: KernelSU temporário; um reboot completo pode fazer o root deixar de estar disponível até ser recuperado.
- `ksud module uninstall ViPER4Android-RE` e os marcadores `disable`, `remove` e `update` não bastaram para limpar as montagens que já estavam ativas.
- O comando de diagnóstico mostrou montagens do ViPER em locais como `/vendor/lib/soundfx`, `/vendor/lib64/soundfx`, `/system/etc/audio_effects.xml` e vários arquivos `audio_effects*.xml`/`.conf` em `/vendor/etc`.
- Foi necessário entrar no namespace de montagens do PID 1 usando o `nsenter` do Toybox Android, que aceita `nsenter -t 1 -m COMMAND...`, mas **não** aceita a sintaxe `-c` do shell diretamente como argumento de `nsenter`.
- As montagens do ViPER foram desmontadas com sucesso. Só depois disso a pasta foi removida. A lista final do KernelSU já não continha `ViPER4Android-RE`, e os outros módulos continuaram listados.

## Quando usar

Use quando:
- o ViPER deixou de funcionar após um reboot ou perda/recuperação do root;
- a desinstalação normal do KernelSU não removeu o módulo completamente;
- o `mountinfo` ainda mostra origens contendo `ViPER4Android-RE`.

Não use como rotina preventiva se a desinstalação normal funcionar. Não use para outros módulos sem adaptar e revisar o procedimento.

## Passo a passo

### 1. Confirme que o root está disponível

Abra o terminal e execute:

```sh
su -c 'id'
```

Continue somente se o comando funcionar com privilégios de root. Se o root ainda não foi recuperado, não tente executar a limpeza.

### 2. Verifique as montagens do ViPER no namespace do sistema

No Android deste caso, o `nsenter` é o Toybox 0.8.12. Para evitar problemas com `-c`, crie um pequeno arquivo de script e execute-o dentro do namespace do PID 1:

```sh
su -c 'cat > /data/local/tmp/viper-check.sh <<'"'"'EOF'"'"'
#!/system/bin/sh
echo "=== Montagens do ViPER no namespace do PID 1 ==="
grep "ViPER4Android-RE" /proc/self/mountinfo
EOF
chmod 700 /data/local/tmp/viper-check.sh
nsenter -t 1 -m /system/bin/sh /data/local/tmp/viper-check.sh'
```

Se não aparecer nenhuma linha, não há montagem com essa identificação nesse namespace. Verifique também o estado do módulo:

```sh
su -c 'ksud module list'
```

### 3. Desmonte somente as montagens que pertencem ao ViPER

O script abaixo lê `/proc/self/mountinfo` já dentro do namespace do PID 1 e tenta desmontar apenas destinos cujos campos de origem (campo 4) apontam para `ViPER4Android-RE`. Ele repete algumas vezes para lidar com camadas sobrepostas.

**Reveja o conteúdo antes de executar.** O script não deve ser alterado para remover montagens de outros módulos.

```sh
su -c 'cat > /data/local/tmp/remove-viper-mounts.sh <<'"'"'EOF'"'"'
#!/system/bin/sh
MI=/proc/self/mountinfo

echo "=== Removendo montagens do ViPER ==="
round=0
while [ "$round" -lt 8 ]; do
  round=$((round + 1))
  found=0

  while IFS= read -r line; do
    set -- $line
    root=$4
    target=$5

    case "$root" in
      */adb/modules/ViPER4Android-RE/*|/adb/modules/ViPER4Android-RE/*)
        found=1
        echo "Desmontando: $target (origem: $root)"
        if umount "$target"; then
          echo "OK"
        else
          echo "FALHOU: $target"
        fi
        ;;
    esac
  done < "$MI"

  [ "$found" -eq 0 ] && break
done

echo "=== Verificação final ==="
grep "ViPER4Android-RE" "$MI"
EOF
chmod 700 /data/local/tmp/remove-viper-mounts.sh
nsenter -t 1 -m /system/bin/sh /data/local/tmp/remove-viper-mounts.sh'
```

Como interpretar:
- Cada linha `OK` significa que a chamada de desmontagem foi aceita.
- A saída de `Verificação final` deve ficar vazia.
- Se aparecer `FALHOU` ou ainda houver linhas de montagens, **não apague a pasta do módulo**. Guarde a saída e investigue o destino que falhou.

### 4. Remova a pasta do módulo só depois de confirmar que não há montagens

Execute:

```sh
su -c 'MOD=/data/adb/modules/ViPER4Android-RE; if grep -q "ViPER4Android-RE" /proc/1/mountinfo; then echo "ERRO: ainda existem montagens; pasta preservada."; else rm -rf "$MOD" && echo "Pasta do ViPER removida."; fi; ksud module list'
```

Esse comando verifica `/proc/1/mountinfo` antes de remover o diretório. A lista do KernelSU não deve mais conter `ViPER4Android-RE`.

### 5. Faça a verificação final

```sh
su -c 'grep "ViPER4Android-RE" /proc/1/mountinfo; test -e /data/adb/modules/ViPER4Android-RE && echo "Pasta ainda existe" || echo "Pasta ausente"; ksud module list'
```

O resultado esperado:
- nenhuma linha de montagem do ViPER;
- `Pasta ausente`;
- `ViPER4Android-RE` ausente na lista de módulos;
- os outros módulos continuam listados.

### 6. Reinstalação e teste

Depois da limpeza:
1. Reinstale o módulo somente se tiver certeza de que o pacote é compatível com sua versão do Android/KernelSU.
2. Siga as instruções do mantenedor do módulo.
3. Teste o áudio antes de adicionar outros módulos de áudio.
4. Evite fazer reboot completo enquanto depender do root temporário, a menos que esteja preparado para recuperar o root novamente.
5. Se o problema reaparecer depois de um reboot, recupere o root e siga este guia desde a verificação de montagens.

## Por que isso foi necessário?

Desativar ou marcar um módulo para remoção não garante que todas as montagens já ativas desapareçam imediatamente. No caso observado, os arquivos do sistema e do fornecedor (`system`/`vendor`) estavam sendo cobertos por montagens originadas na árvore do ViPER. Apagar a pasta antes de desmontar essas camadas pode falhar com `Device or resource busy` ou deixar o estado difícil de diagnosticar.

O `nsenter -t 1 -m` permite executar o script no namespace de montagens do processo 1, onde estavam as montagens problemáticas. O script identifica a origem específica do ViPER e tenta desmontar apenas essas entradas. Depois que a verificação fica vazia, a pasta pode ser removida.

## Limitações e cuidados

- Este procedimento foi validado no estado descrito nesta documentação; não é uma garantia para todas as versões de Android, KernelSU, Toybox ou ViPER.
- Os campos de `/proc/*/mountinfo` têm formato técnico. Se a saída for diferente, pare e revise em vez de adaptar às cegas.
- Não use `umount -a`, não desmonte diretórios genéricos de áudio sem confirmar a origem e não remova a pasta antes da verificação.
- O comando `rm -rf` é destrutivo. Neste procedimento, ele é usado somente para `/data/adb/modules/ViPER4Android-RE`, depois da verificação de montagens.
- Guarde a saída do terminal se qualquer desmontagem falhar.
- A remoção do módulo pode deixar o processamento de áudio diferente até reiniciar serviços ou reinicializar o aparelho. Como o root é temporário, considere esse risco antes de qualquer reinicialização.
- Os comandos não alteram intencionalmente os outros módulos. Sempre confira a lista do KernelSU antes e depois.

## Resumo rápido

1. Recuperar o root e confirmar `su -c 'id'`.
2. Inspecionar as montagens via `nsenter -t 1 -m`.
3. Executar `remove-viper-mounts.sh`.
4. Confirmar que `grep "ViPER4Android-RE" /proc/1/mountinfo` não retorna nada.
5. Executar o comando de remoção da pasta e verificar `ksud module list`.
6. Só então reinstalar/testar o ViPER, se desejado.
