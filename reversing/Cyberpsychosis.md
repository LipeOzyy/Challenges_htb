# CYBERPSYCHOSIS

Este post detalha a resolução de um desafio de engenharia reversa focado em um rootkit de kernel Linux. Nosso objetivo era desarmar o módulo e encontrar a flag oculta.

## Analise inicial: O que temos?

O primeiro passo foi identificar o binario suspeito:

[](/reversing/img/FILE_.png)
[](/reversing/img/analises.png)

Esta análise estática confirmou nosso plano, o próximo passo é descompilar diamorphine_init para ver como e quais hooks são instalados e, em seguida, analisar as funções hacked_*.
O binario funciona interceptando a System Call Table do Kernel

## Usando o Ghidra

Abri o Ghidra para analisar melhor o binario.

Dentre as funções que me chamaram a atenção, analisei diamorphine_init.
[](/reversing/img/init.png)