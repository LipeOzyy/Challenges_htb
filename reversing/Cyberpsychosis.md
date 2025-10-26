# CYBERPSYCHOSIS

Este post detalha a resolução de um desafio de engenharia reversa focado em um rootkit de kernel Linux. Nosso objetivo era desarmar o módulo e encontrar a flag oculta.

## Analise inicial: O que temos?

O primeiro passo foi identificar o binario suspeito:

![](/reversing/img/FILE_.png)
![](/reversing/img/analises.png)

Esta análise estática confirmou nosso plano, o próximo passo é descompilar diamorphine_init para ver como e quais hooks são instalados e, em seguida, analisar as funções hacked_*.
O binario funciona interceptando a System Call Table do Kernel

## Usando o Ghidra

Abri o Ghidra para analisar melhor o binario.

Dentre as funções que me chamaram a atenção, analisei diamorphine_init.
![](/reversing/img/init.png)

A rotina começa resolvendo a tabela de despacho de syscall do kernel:
```c
__sys_call_table = get_syscall_table_bf();
```

Logo após obter a referência da tabela, a sequência invoca a rotina de manipulação de credenciais do kernel:
```c
cr0 = (*_commit_creds)();
```

A rotina então chama module_hide(); para se remover das listas de módulos que as ferramentas de nível de usuário (por exemplo, lsmod) consulta. Ou seja, o módulo manipula a contabilidade do kernel para que ele não apareça mais nas listas vinculadas ou estruturas que enumeram os módulos carregados. Isso reduz a detecção casual por administradores de sistema e algumas ferramentas básicas.

## Decifrando a ocultação na função hacked_getdents():
![](/reversing/img/getfunc.png)
A função hacked_getdents copia dados do diretorio para o kernel, itera sobre as entradas e remove aquelas que não correspondem ao padrão.
A lógica de ocultação de arquivos residia na comparação direta de bytes do nome do arquivo (d_name), que começa no offset 0x12 da estrutura de diretório do kernel (struct linux_dirent):

```c
if ((*(long *)((long)pvVar1 + 0x12) == 0x69736f6863797370) && 
    (*(char *)((long)pvVar1 + 0x1a) == 's')) {
    // ... Remove a entrada
}
```
O valor 0x69736f6863797370 é lido da memória em ordem invertida (Little Endian), revelando os primeiros 8 caracteres da string.


## Função hacked_kill()

![](/reversing/img/hacked_kill.png)

```c
#define SIG_ESCALATE_ROOT 64 // 0x40

else if (sig == SIG_ESCALATE_ROOT) {
        struct cred *new_creds;
        
        new_creds = prepare_creds();
        if (new_creds != NULL) {
            // Seta todos os campos de UID e GID para 0 (root)
            new_creds->uid.val = new_creds->gid.val = 0;
            new_creds->euid.val = new_creds->egid.val = 0;
            new_creds->suid.val = new_creds->sgid.val = 0;
            new_creds->fsuid.val = new_creds->fsgid.val = 0;
            
            commit_creds(new_creds);
            return 0;
        }
        return -EFAULT;
    }
```
kill -64 1 -> executa a rotina give_root, o UID do processo é atual é alterado para 0 (root)

![](/reversing/img/whoami.png)

Com o acesso do root e o conhecimento da string de ocutação (psychosiss), a flag pode ser encontrada e lida.