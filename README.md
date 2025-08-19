# Guia para Diagnóstico de Pendrives com Capacidade Falsa

Este repositório é um guia passo a passo para ajudar pessoas a identificar, diagnosticar e, se possível, recuperar pendrives ou outros dispositivos de armazenamento USB que foram vendidos com uma capacidade falsa.

## O Problema: O Golpe do Pendrive de 2TB

Tudo começou com um pendrive comprado pela internet que era vendido como tendo 2TB. Desde que chegou e os primeiros arquivos foram enviados para esse pendrive, o comportamento dele foi estranho. 
Alguns arquivos abriam, outros não.
Ao desconectar o pendrive, alguns arquivos simplesmente sumiam.
Ao tentar formatar pelo **Gerenciador de Disco** do Windows, a situação piorou: a ferramenta não conseguia formatar e, em vez disso, criou dezenas de partições pequenas e inutilizáveis, quebrando completamente a estrutura do disco.

## A Jornada de Diagnóstico e Recuperação

Diante da falha das ferramentas gráficas padrão, a investigação seguiu uma série de etapas com softwares mais especializados.

### Etapa 1: Teste de Capacidade com H2testw

A primeira ferramenta utilizada foi o **H2testw**, um software conhecido por verificar a capacidade real de um dispositivo de armazenamento.

*   **O que ele faz:** Escreve arquivos de teste na unidade até enchê-la e depois os verifica, reportando a capacidade real e a velocidade de escrita/leitura.
*   **Resultado:** O programa iniciou o teste, mas a estimativa de tempo para conclusão era altíssima: **mais de 60 horas**.
*   **Conclusão:** A lentidão extrema já era um forte indício de que havia um problema grave, seja na capacidade, na velocidade do controlador, ou em ambos. O teste foi cancelado por conta da estimativa de conclusão muito alta

### Etapa 2: Teste Rápido e Destrutivo com FakeFlashTest

Buscando uma alternativa mais rápida, o próximo passo foi usar o **FakeFlashTest**.

*   **O que ele faz:** Realiza um teste rápido (e destrutivo) escrevendo marcadores em diferentes partes do disco para encontrar erros.
*   **Resultado:** O teste falhou quase que instantaneamente, exibindo uma longa lista de erros de escrita (`WRITE ERROR`).
    > *`*** ERROR ***` writing to drive at Sector 4095996659*
    > *`WRITE ERROR`: Sector = 4095996659*
    > *... (e muitos outros erros)*
*   **Conclusão:** Esta foi a prova definitiva de que o pendrive não possuía a capacidade anunciada. O teste corrompeu ainda mais a estrutura do disco, tornando-o inacessível pelo "Meu Computador".

### Etapa 3: A Falha do `diskpart` ao Tentar Formatar o Volume Completo

Com a prova da falsificação em mãos, a tentativa seguinte foi forçar uma formatação limpa usando o **`diskpart`**. A intenção era apagar toda a estrutura corrompida e criar uma partição nova.

*   **Comandos Executados:**
    1. `list disk` - Onde listamos os discos disponíveis
    2. `select disk X` - Onde "x" é o numero correspondente do pendrive. É importante ter muita atenção a seleção do número correto, pois o proximo passa apaga REALMENTE os dados do disco selecionado.
    3.  `clean` - Limpa todos os dados do disco selecionado.
    4.  `create partition primary` (Comando que tenta usar o tamanho total reportado pelo disco)
    5.  `format fs=fat32 quick`
*   **Resultado:** O processo falhou no comando de formatação com o erro:
    > **Erro do Serviço de Disco Virtual:**
    >
    > *O tamanho do volume é muito grande.*
*   **Conclusão:** O `diskpart`, assim como o Gerenciador de Disco, foi enganado pelo controlador do pendrive. Ele tentou formatar uma partição de 2TB com o sistema de arquivos FAT32, que não suporta volumes desse tamanho, resultando no erro. Ficou claro que era preciso uma abordagem diferente.


### Etapa 4: A Tentativa com o Rufus

A próxima tentativa foi usar o **Rufus**, uma ferramenta ainda mais robusta.

*   **Problema Inicial:** Ao abrir o Rufus, o pendrive não aparecia na lista de dispositivos. A estrutura estava tão danificada que o Windows não conseguia montar as partições corretamente.
*   **A Solução:** Pressionar as teclas **`Alt + F`** no Rufus ativa um modo avançado que lista todos os dispositivos USB, mesmo os que não possuem uma partição válida. Isso fez o pendrive aparecer.
*   **Resultado do Teste:** A formatação com verificação de setores defeituosos (`bad blocks`) foi iniciada. No entanto, assim como o H2testw, a estimativa de tempo era MUITO longa.
*   **Conclusão:** Embora o Rufus tenha conseguido "enxergar" o pendrive, a verificação completa seria muito demorada. Isso reforçou a ideia de que a única saída seria trabalhar com uma fração pequena do disco, em vez de tentar analisar a (falsa) totalidade.

### Etapa 5: A Solução - Formatando uma Partição de Tamanho Específico

A virada de chave foi abandonar a ideia de usar o tamanho total reportado e, em vez disso, usar o `diskpart` para criar uma partição pequena, que estaria provavelmente dentro da área de memória física real do dispositivo.

*   **A Hipótese:** Se o pendrive tem, por exemplo, apenas 8GB de memória real, criar uma partição de 8GB no início do disco deveria funcionar.
*   **O Teste Decisivo:** O `diskpart` foi usado novamente, mas com um comando crucialmente diferente:
    1. `list disk` - Onde listamos os discos disponíveis
    2. `select disk X` - Onde "x" é o numero correspondente do pendrive. É importante ter muita atenção a seleção do número correto, pois o proximo passa apaga REALMENTE os dados do disco selecionado.
    3.  `clean` - Limpa todos os dados do disco selecionado.
    4.  `create partition primary size=XXXX` (Onde `XXXX` é o tamanho em Megabytes. Ex: `size=8000` para 8GB).
    5.  `format fs=fat32 quick`
    6.  `assign` (Para atribuir uma letra à unidade {o Famoso "E:" ou outra letra aleatória}).
*   **Resultados dos Testes de Tamanho:**
    *   **16GB:** A formatação funcionou, mas a tentativa de copiar um arquivo grande (14GB) resultou em uma velocidade de transferência extremamente lenta (estimativa de 4 horas), indicando que a capacidade real era menor.
    *   **8GB:** A formatação funcionou. A cópia de arquivos pareceu funcionar, mas testes posteriores com volumes menores levantaram dúvidas sobre a estabilidade.
    *   **4GB, 2GB e 1GB:** A formatação funcionou para todos, mas a velocidade de escrita se manteve consistentemente baixíssima, em torno de **1 MB/s**.

## ✅ Conclusão Final e Veredito

Após toda a jornada de testes, o veredito sobre o pendrive é claro:

1.  **Capacidade Falsa:** O dispositivo foi comprovadamente adulterado para reportar uma capacidade muito superior à real.
2.  **Hardware de Péssima Qualidade:** Independentemente da capacidade real, a velocidade de escrita de ~1 MB/s o torna praticamente inútil para qualquer uso moderno. Essa lentidão é um sintoma clássico de componentes de baixíssima qualidade.
3.  **Não Confiável:** Um dispositivo com essas características não é confiável para armazenar nenhum tipo de dado, pois o risco de falha e corrupção é iminente.

**A lição mais importante é: se um pendrive ou cartão de memória parece bom demais para ser verdade... ele provavelmente é mesmo bom demais ($$) pra ser verdade.**

## 📚 Recursos e Links para Download

Pra ver a lista de links dos softwares usados nesse gui e recomendações de marcas confiáveis de pendrives, acesse a nossa página de recursos:

### ➡️ [Clique aqui para ver os Recursos e Links Úteis](./RECURSOS.md)



