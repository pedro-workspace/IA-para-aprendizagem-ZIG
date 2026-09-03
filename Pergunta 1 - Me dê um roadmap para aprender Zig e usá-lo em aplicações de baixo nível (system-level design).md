# Roadmap de Aprendizado: Zig para Sistemas e Baixo Nível

Este guia foi elaborado para direcionar o aprendizado da linguagem de programação **Zig** [10] focado no desenvolvimento de sistemas de baixo nível e design de software robusto (*system-level design*), aproveitando ao máximo o controle de hardware, previsibilidade e metaprogramação que a linguagem oferece.

---

## Fase 1: Introdução e Filosofia de Design

O primeiro passo é compreender que o Zig foi desenhado como uma alternativa robusta e sem compromissos para substituir C [528, 562]. Ele prioriza a legibilidade do código, a comunicação precisa com o compilador e a ausência de comportamento oculto [10, 527].

1. **Rigor e Controle de Erros Estáticos**:
   - Desenvolva seus primeiros programas entendendo a rigidez do compilador. No Zig, **variáveis locais ou importações declaradas que não forem utilizadas geram erros de compilação** [14, 569]. Essa escolha previne que bugs silenciosos cheguem ao ambiente de execução [569].

2. **Uso de Recursos Offline e Oficiais**:
   - **Documentação de Página Única**: Toda a referência da linguagem é mantida em uma única página HTML, projetada para ser offline-friendly e altamente pesquisável [11]. Todos os exemplos dela são testados e compilados junto com os testes do próprio compilador para garantir exatidão [11].
   - **Biblioteca Padrão Local**: Use o utilitário `zig std` no seu terminal para renderizar a documentação da biblioteca padrão localmente através de um servidor web no seu computador [12].

3. **Prática Prática Guiada (Ziglings)**:
   - Comece corrigindo pequenos problemas de código com o projeto comunitário **Ziglings** (criado por Dave Gower e mantido por Chris Bosch) [572, 573]. Essa é a melhor e mais recomendada forma prática de aprender a sintaxe e os recursos nativos do Zig incrementalmente [572, 573].

---

## Fase 2: Gerenciamento Explícito de Memória

Diferente de linguagens modernas como Rust e C++, o Zig não utiliza desalocação implícita baseada em escopo (como RAII) [566]. Ao contrário de Go, Java ou C#, ele **não possui coletor de lixo (Garbage Collector)** [530, 566]. A alocação de memória é totalmente explícita [457, 566].

1. **Convenção de Alocação**:
   - Por convenção, **não há um alocador padrão global** [457]. Qualquer função ou estrutura que necessite de memória dinâmica deve receber explicitamente um argumento do tipo `Allocator` [457].

2. **Tipos de Alocadores a Dominar**:
   - `FixedBufferAllocator`: Use quando o tamanho máximo de bytes necessários em tempo de execução for predefinido e puder ser alocado em um buffer estático [459, 460].
   - `ArenaAllocator`: Ideal para agrupar várias alocações temporárias que compartilham o mesmo ciclo de vida (como o processamento de um request de rede ou um frame gráfico) e liberá-las todas de uma única vez chamando `arena.deinit()` [461, 566].

3. **Design de Ultra-Baixa Latência (Pré-Alocação Total)**:
   - Siga o padrão de projetos de alto desempenho (como o banco de dados financeiro *Tiger Beetle*) [533]: **pré-aloque toda a memória que a aplicação precisará logo na sua inicialização e evite qualquer alocação dinâmica em runtime** [533, 534]. Isso mantém a latência de execução previsível e livre de surpresas [534].

---

## Fase 3: Ponteiros, Slices e Modos de Segurança

Trabalhar em sistemas de baixo nível exige interagir diretamente com endereços de memória, mas o Zig faz isso com salvaguardas modernas de segurança lógica.

1. **Tipos de Ponteiros do Zig**:
   - `*T`: Ponteiro para um único item [82]. Permite ler e escrever no valor referenciado por meio da desreferenciação explícita `ptr.*` [82].
   - `[*]T`: Ponteiro para múltiplos itens de tamanho desconhecido, aceitando aritmética de ponteiros e indexação via colchetes `ptr[i]` [82].
   - `[]T` (*Slice*): Uma estrutura de dados fat-pointer composta por um ponteiro muitos-itens `[*]T` e um comprimento `len` conhecido em tempo de execução [83]. Possui **verificação automática de limites de índice (*bounds checking*)** para capturar acessos ilegais [86, 101].

2. **Modos de Compilação e Otimização**:
   - Zig dispõe de quatro modos de compilação: `Debug`, `ReleaseFast`, `ReleaseSafe` e `ReleaseSmall` [405].
   - Nos modos de segurança (`Debug` e `ReleaseSafe`), o compilador insere **verificações de segurança em tempo de execução** para capturar comportamentos ilegais, como estouro de inteiros (*integer overflow*) e índices fora de limites [30, 409].
   - Use o recurso `@setRuntimeSafety(bool)` para habilitar ou desabilitar essas verificações de segurança em blocos de código críticos sob demanda [373, 409].

---

## Fase 4: Programação Próxima ao Hardware (MMIO, Bits e Assembly)

Esta fase foca no controle cirúrgico dos recursos físicos do processador e barramentos da placa.

1. **Modelagem de Registradores com `packed struct`**:
   - Use structs marcadas com o modificador `packed` para representar pacotes de rede ou mapear registradores físicos de hardware. Ao contrário de structs normais, as packed structs possuem layout de memória garantido idêntico ao de seu inteiro de suporte (*backing integer*) [115].
   - É possível utilizar variáveis com bit-widths arbitrários e precisos (como `u3` ou `u5`) como campos [120, 121, 123].

2. **MMIO e Ponteiros `volatile`**:
   - Ao interagir com registradores de entrada/saída mapeados em memória (MMIO), qualifique seus ponteiros com o modificador `volatile` [92]. Isso garante que todas as operações físicas ocorram na ordem exata prescrita, impedindo que o otimizador as remova ou reordene [92].
   - **Cuidado com Alterações Parciais**: Modificações de campos de packed structs não são atômicas [128]. Em MMIO, em vez de modificar campos voláteis diretamente, monte o valor inteiro completo em memória regular e, em seguida, grave-o de uma vez no ponteiro volatile [128, 129].

3. **Assembly Inline e Global**:
   - Utilize a palavra-chave `asm` (e `asm volatile` para prevenir deleção se o resultado não for usado) para injetar instruções de máquina diretamente no fluxo de execução, detalhando restrições de entrada, saída e clobbers [304, 305].
   - Utilize assembly global dentro de blocos `comptime` no nível de namespace quando precisar expor símbolos globais diretamente para o linker [308, 309].

---

## Fase 5: Interoperabilidade Perfeita com C

A interoperabilidade nativa com C é um dos pilares do Zig, que pode traduzir, compilar e linkar código C sem qualquer overhead.

1. **Tipos Primitivos do C ABI**:
   - O Zig disponibiliza tipos de dados nativos perfeitamente compatíveis com a ABI do C (como `c_int`, `c_char`, `c_ulong`) [21, 483, 484].

2. **Tradução Automática de Headers**:
   - Use o utilitário CLI `zig translate-c` para converter arquivos `.h` de C automaticamente para código equivalente nativo em Zig [484, 485].

3. **Linkagem e Assinaturas Externas**:
   - Integre códigos e bibliotecas C declarando as assinaturas de funções com o modificador `extern` [46, 196, 514].
   - Exporte funções escritas em Zig para serem chamadas por C marcando-as como `export` e usando a convenção de chamada C ABI (`callconv(.c)`) [154, 514].

4. **Ponteiros de Tipos Opacos**:
   - Use tipos declarados como `opaque {}` para lidar com ponteiros de estruturas do C cujo layout de memória não é exposto ou detalhado, mantendo total segurança de tipos [154, 487].

---

## Fase 6: Metaprogramação Segura com `comptime`

A metaprogramação no Zig dispensa sistemas complexos de macros ou pré-processadores externos. Em vez disso, ela é feita utilizando a própria sintaxe do Zig executada em tempo de compilação.

1. **Genéricos através de Parâmetros Comptime**:
   - Genéricos são implementados passando tipos comuns como argumentos normais de funções, marcando esses parâmetros com o qualificador `comptime` [275, 295]. O compilador gera código especializado altamente performático usando duck typing em tempo de compilação [275, 276, 278].

2. **Unrolling e Inicialização Complexa**:
   - Use variáveis marcadas com `comptime var` e laços `inline while` ou `inline for` para unrollar loops fisicamente ou computar tabelas estáticas de constantes matemáticas direto no binário final [52, 281, 282, 284].
