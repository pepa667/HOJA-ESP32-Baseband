# Analise do fw_v2 e proposta de novo projeto usando HOJA-baseband como componente

## Objetivo deste documento

Este documento registra uma analise dos projetos:

- `fw_v2/blu-switch`
- `fw_v2/blu-common`

e traduz essa leitura em uma proposta tecnica para criar um novo projeto usando o `HOJA-baseband` como base de componente, sem repetir cegamente a organizacao atual do `fw_v2`.

Ao longo deste documento, quando eu digo `HOJA-baseband`, estou me referindo especificamente ao repositorio upstream `HandHeldLegend/HOJA-ESP32-Baseband`.

O foco aqui nao e apenas descrever arquivos, mas responder estas perguntas praticas:

- como esses dois projetos realmente se encaixam
- qual deles faz leitura de hardware e qual deles faz protocolo
- como desenhar a integracao do `HOJA-baseband` de forma limpa e reutilizavel
- como desenhar um projeto novo de forma mais limpa
- qual e o impacto da sua ideia de hardware, incluindo o arranjo de botoes e o uso de matriz com diodos

---

## Atualizacoes de hardware e perguntas respondidas

As decisoes e duvidas abaixo refinam o desenho do produto e afetam diretamente a arquitetura recomendada.

### Botoes extras: `SHIFT` e `PROTOCOL`

Adicionar `SHIFT` e `PROTOCOL` faz sentido e melhora muito a usabilidade de um controle derivado da shell do GBA.

Minha leitura e esta:

- `SHIFT` deve continuar sendo um modificador de input puro
- `PROTOCOL` deve virar um botao de sistema, voltado a troca de modo, pairing e funcoes administrativas

Separar esses dois papeis e a decisao certa. Misturar `SHIFT` com mudanca de protocolo aumentaria a chance de erro e deixaria a UX confusa.

### Sugestao de comportamento para o botao `PROTOCOL`

Uma politica simples e forte seria:

- toque curto: alterna protocolo salvo
- pressao longa: entra em modo de pareamento
- `SHIFT + PROTOCOL`: reseta pareamento do protocolo atual
- `SHIFT + segurar PROTOCOL no boot`: modo especial, como recovery ou firmware slot alternativo

Isso elimina a necessidade de um botao dedicado de pair/reset, e reaproveita bem um unico botao de sistema.

### Mais ideias de botao ou interface, se voce quiser margem futura

Eu nao colocaria muitos botoes extras alem de `SHIFT` e `PROTOCOL`, mas consideraria uma destas opcoes discretas:

- pads de teste para `BOOT` e `RESET`, acessiveis por pogo pin ou solda temporaria
- um micro tact escondido interno apenas para manutencao, nao para o usuario final
- uma combinacao de boot com `SHIFT` e `PROTOCOL` para forcar modo de recuperacao

Isso ajuda muito na bancada sem poluir a frente do produto.

### Sem botao dedicado de power, sleep e awake

Usar a chave original do GBA como chave principal de energia e uma decisao coerente para este projeto.

Para um produto compacto, isso tem varias vantagens:

- interface familiar
- menos componentes externos
- menos complexidade de UX
- desligamento fisico real

Minha ressalva tecnica e esta:

- a chave deve cortar a alimentacao do sistema de forma correta no caminho de potencia
- ela nao deve contornar protecao de bateria, circuito de carga ou power-path, se esses blocos existirem

Em outras palavras: usar a chave original como liga/desliga principal e bom; usar a chave para "burlar" a gestao correta de energia da bateria nao e uma boa ideia.

### E necessario manter `sleep` e `awake` em firmware?

Nao necessariamente.

Se houver desligamento fisico pela chave, voce pode simplificar o firmware e tratar `sleep/wake` como opcional, nao como requisito de primeira versao. Isso reduz complexidade.

O que eu manteria mesmo sem botao dedicado:

- pairing por combinacao
- reset de pareamento por combinacao
- alguma forma de recovery ou modo de manutencao

### Troca do display por painel impresso com LEDs

Trocar o display por painel impresso com LEDs me parece uma decisao muito mais alinhada com as restricoes de espaco, custo e consumo.

Isso tambem combina melhor com a shell do GBA, porque permite manter uma frente limpa e ainda assim expor estado do sistema.

### Se eu comecar com GPIO direto, depois fica muito dificil migrar para matriz de inputs?

No software, nao precisa ficar dificil.

No hardware, pode ficar bem mais dificil se a placa nao for planejada com isso em mente.

Resumo realista:

- dificuldade de firmware: baixa a moderada, se voce abstrair direito desde o inicio
- dificuldade de PCB: moderada a alta, se hoje tudo for roteado como GPIO dedicado sem pensar em linhas e colunas

Entao a resposta correta e: a migracao futura para matriz e viavel, mas so sera tranquila se voce preparar a arquitetura agora.

### Como deixar isso preparado desde o inicio

No firmware:

- crie uma interface logica unica de input, por exemplo `input_state_t`
- faca a camada superior consumir apenas estado logico, nunca GPIO bruto
- isole a leitura fisica em um backend, por exemplo `input_gpio.c` ou `input_matrix.c`

Na placa:

- reserve a possibilidade de reorganizar os botoes em linhas e colunas
- considere jumpers de 0 ohm ou footprint alternativo para reconfigurar sinais
- evite assumir que cada botao tera sempre um GPIO exclusivo ate a versao final

Conclusao pratica:

- se a abstracao for bem feita, mudar de GPIO direto para matriz nao quebra o projeto inteiro
- o que mais sofre nao e o app nem o protocolo, e sim a camada de leitura fisica e a PCB

### A mesma logica vale para os LEDs?

Sim.

Para os LEDs, a regra e praticamente a mesma:

- se voce codificar tudo como `gpio_set_level(led_x, ...)` espalhado pelo projeto, a migracao depois fica chata
- se voce criar uma camada `status_leds` desde o inicio, a mudanca fica controlada

Exemplo de interface boa:

- `status_leds_set_protocol(protocol_id)`
- `status_leds_set_pairing(active)`
- `status_leds_set_battery(level)`
- `status_leds_set_error(code)`

Por baixo dessa interface, mais tarde voce pode trocar:

- GPIO direto
- matriz multiplexada
- shift register
- expansor I2C
- driver dedicado de LED

Conclusao pratica:

- para inputs, abstrair cedo ajuda muito
- para LEDs, abstrair cedo ajuda ainda mais

### Sugestoes para os LEDs informativos

Como voce vai usar painel impresso, eu seguiria uma filosofia de poucos estados, mas muito claros.

Minha sugestao principal e separar os LEDs por funcao, nao por enfeite.

#### Conjunto minimo recomendado

1. LED de protocolo
  mostra qual protocolo esta ativo

2. LED de conexao
  mostra buscando, pareando e conectado

3. LED de bateria
  mostra bateria baixa ou estado de carga

Isso ja da uma base muito boa.

#### Forma simples de representar protocolo

Voce pode usar:

- 3 LEDs em codigo binario para ate 7 protocolos
- ou 4 LEDs dedicados, um por familia principal

Se quiser algo mais legivel para usuario final, eu prefiro 4 indicadores nomeados no painel, por exemplo:

- `SW`
- `XI`
- `DI`
- `GEN`

Se quiser algo mais flexivel e economico em area, 3 LEDs em binario funcionam bem.

#### Forma simples de representar conexao

Um unico LED de estado pode cobrir quase tudo com padroes de blink:

- piscando lento: procurando host
- piscando rapido: pairing
- aceso fixo: conectado
- pulso curto periodico: ligado, mas ocioso

#### Forma simples de representar bateria

Decisao atual do projeto:

- usar apenas 1 LED vermelho de power
- o LED pisca e aumenta a frequencia conforme a bateria abaixa

Sugestao de mapeamento simples de frequencia:

- bateria alta: pulso lento
- bateria media: pulso moderado
- bateria baixa: pulso rapido
- bateria critica: pulso muito rapido e janela curta de desligamento seguro

Esse esquema e classico, barato e combina com a ideia de painel impresso minimalista.

#### Sugestao forte para reduzir GPIO

Se os LEDs passarem de uns 3 ou 4 canais, eu comecaria a considerar logo um driver simples em vez de gastar GPIO demais.

Opcoes praticas:

- `74HC595`, se quiser custo baixo e simplicidade
- um expansor I2C pequeno, se ja existir barramento e a contagem de LEDs crescer
- multiplexacao propria, se quiser economizar pinos e aceitar um pouco mais de firmware

Para primeira versao, eu faria assim:

- ate 3 ou 4 LEDs: GPIO direto
- acima disso: pensar logo em `74HC595` ou outra solucao parecida

### Minha recomendacao objetiva de hardware para esta fase

Se eu tivesse que escolher um caminho agora, seria este:

- manter `SHIFT`
- adicionar `PROTOCOL`
- usar a chave original do GBA como power principal
- remover display da primeira versao
- usar painel impresso com poucos LEDs de status
- manter LED de power unico (vermelho) com frequencia de pisca proporcional ao nivel de bateria
- implementar camada de abstração para inputs e LEDs desde o inicio
- começar com GPIO direto se isso acelerar prototipo
- mas deixar a arquitetura pronta para migrar depois para matriz de inputs e driver de LEDs
- usar firmware unico na primeira fase, com arquitetura preparada para expansao futura

Essa combinacao preserva simplicidade agora, sem te prender num beco sem saida mais tarde.

---

## Resumo executivo

O `fw_v2` mostra uma arquitetura diferente da que aparece em `fw_v3`.

No `fw_v2`, o projeto principal `blu-switch` nao usa o `HOJA-baseband` diretamente como bloco ativo do firmware. Na pratica, ele usa:

- `blu-common` para leitura de hardware
- `HOJA-LIB-ESP32` como camada de frontend/backend e orquestracao dos cores
- `BluControl-SDK-ESP32` para troca de modo via particoes OTA

Essa e a conclusao mais importante desta analise.

Ou seja:

- `fw_v2` nao e um exemplo real de projeto que ja usa o `HOJA-baseband` como componente reutilizavel
- `fw_v2` e um exemplo de projeto que usa `HOJA-LIB-ESP32` como framework e `blu-common` como camada de hardware
- para criar um novo projeto com `HOJA-baseband` como componente, sera preciso desenhar uma integracao propria

---

## O que cada projeto faz

## `blu-switch`

`blu-switch` e o projeto de aplicacao.

Ele faz principalmente:

- inicializacao do app
- registro de callbacks do HOJA
- escolha do core ativo
- inicializacao da camada de hardware
- troca de modo por particoes OTA

O ponto de entrada em `main/main.c` mostra claramente esse papel.

Fluxo observado:

1. registra callbacks com `hoja_register_button_callback`, `hoja_register_analog_callback`, `hoja_register_event_callback` e `hoja_register_rumble_callback`
2. inicializa energia e hardware com `blu_energy_init()` e `blu_init_hardware()`
3. inicializa o controle de modo com `blucontrol_mode_init(true)`
4. sobe o framework com `hoja_init()`
5. seleciona o core com `hoja_set_core(HOJA_CORE_NS)`
6. escolhe o subcore com `core_ns_set_subcore(HOJA_CONTROL_TYPE)`
7. inicia o core em loop ate sucesso com `hoja_start_core()`

Conclusao pratica:

`blu-switch` e um shell de aplicacao em cima do ecossistema HOJA-LIB, nao em cima do `HOJA-baseband` puro.

## `blu-common`

`blu-common` e a camada de hardware reaproveitavel.

Ele contem:

- leitura de botoes GPIO
- leitura de sticks analogicos
- leitura de sticks por botoes digitais
- leitura de sticks N64
- leitura de triggers analogicos
- controle de energia
- configuracao por `menuconfig`

Seu papel e alimentar a camada superior com estados fisicos do controle.

Conclusao pratica:

`blu-common` e o HAL do projeto, ou seja, a camada de adaptacao entre o hardware real e a API superior.

---

## Arquitetura real do `fw_v2`

O `fw_v2` pode ser entendido como uma pilha de 4 niveis.

### Camada 1: aplicacao

Projeto:

- `blu-switch/main`

Responsabilidades:

- sequencia de boot
- callbacks
- escolha de core
- politica de modo

### Camada 2: hardware

Projeto:

- `blu-common/blu-components`

Responsabilidades:

- GPIO
- ADC
- leitura de botoes
- leitura de sticks
- trigger analogico

### Camada 3: framework HOJA

Projeto:

- `blu-switch/components/HOJA-LIB-ESP32`

Responsabilidades:

- inicializacao do framework
- estado global de botoes e analogicos
- spawn de task de leitura
- selecao de cores
- backend de protocolos e consoles

### Camada 4: troca de firmware por particao

Projeto:

- `blu-switch/components/BluControl-SDK-ESP32`

Responsabilidades:

- troca entre particoes OTA
- leitura de botoes de troca de modo
- LEDs de indicacao de modo

---

## Descobertas principais

## 1. O `fw_v2` usa `HOJA-LIB-ESP32`, nao o `HOJA-baseband`, como framework principal

O projeto observado sobe via:

- `hoja_init()`
- `hoja_set_core()`
- `hoja_start_core()`

que pertencem ao `HOJA-LIB-ESP32`, nao ao `HOJA-baseband` puro.

Conclusao:

- o `fw_v2` e organizado em torno do `HOJA-LIB-ESP32`
- ele nao serve como exemplo direto de integracao modular do `HOJA-baseband`

## 2. O `fw_v2` separa relativamente bem aplicacao, hardware e framework

O arquivo `HOJA-LIB-ESP32/hoja_frontend.c` mostra um frontend que:

- inicializa NVS
- sobe `hoja_button_task`
- registra callbacks
- escolhe e inicia cores como `HOJA_CORE_NS`, `HOJA_CORE_BT_DINPUT` e `HOJA_CORE_BT_XINPUT`

Isso muda muito a leitura do projeto.

No `fw_v2`, a arquitetura ja esta orientada para:

- callback de input
- estado global compartilhado
- cores configuraveis

e nao para o modelo I2C slave do `HOJA-baseband` standalone.

## 3. `blu-common` assume hardware direto, nao matriz escaneada

`blu-hardware.c` mostra claramente uma logica de leitura de botoes por GPIO direto.

Caracteristicas observadas:

- cada botao recebe uma string de GPIOs vinda do `menuconfig`
- pode haver mais de um GPIO por botao
- o estado final exige leitura direta desses GPIOs

Isso nao e um scanner de matriz.

O comportamento atual e este:

- configura GPIOs individualmente como entrada
- le nivel logico direto
- monta o estado de cada botao

Ou seja, a camada atual nao foi escrita para um teclado matricial com linhas e colunas escaneadas.

## 4. Seu hardware sugerido pelas imagens parece mais proximo de matriz escaneada com diodos

Pelos esquemas anexados, aparecem sinais como:

- `SCN_PORT_A`
- `SCN_PORT_B`
- `SCN_PORT_C`
- `SCN_PORT_D`

e uma disposicao com chaves e diodos.

Isso indica um modelo mais proximo de:

- linhas de varredura
- grupos de botoes isolados por diodos
- leitura multiplexada ou matricial

Conclusao pratica:

se o seu hardware final seguir essa direcao, o `blu-common` atual nao resolve sozinho. Sera necessario criar uma nova camada de scanner de matriz.

## 5. O `fw_v2` ja implementa duas ideias importantes para o seu projeto

### Shift minimalista

No `main/main.c` de `blu-switch`, quando `CONFIG_BLUCONTROL_BUTTONS_MINIMAL_YES` esta ativo, o projeto implementa:

- `A -> X`
- `B -> Y`
- `L -> ZL`
- `R -> ZR`
- `START -> HOME`
- `SELECT -> CAPTURE`

e, no caso do stick por botoes, tambem permite:

- normal: stick digital gera eixo analogico
- com shift: stick digital vira D-pad

Isso esta muito alinhado com a sua ideia.

### Troca de modo por OTA

O `BluControl-SDK-ESP32` usa particoes OTA para alternar entre imagens diferentes.

A tabela de particoes em `blu-common/partitions.csv` ja mostra essa intencao:

- `ota_mode`
- `switch_mode`
- `generic_mode`

E `blucontrol_mode.c` troca a particao de boot com:

- `esp_ota_set_boot_partition()`
- `esp_restart()`

Conclusao pratica:

o `fw_v2` nao faz troca elegante de protocolo dentro do mesmo runtime. Ele troca de firmware por reboot.

---

## O que esses projetos ensinam para um projeto novo

## Licao 1: separar hardware de protocolo funciona bem

Esse e o melhor aspecto do `fw_v2`.

`blu-common` le hardware.

`HOJA-LIB-ESP32` cuida do framework e dos cores.

`main.c` apenas orquestra.

Essa separacao e boa e deve ser preservada.

## Licao 2: `HOJA-baseband` nao esta pronto para ser "componente" plug-and-play

O `HandHeldLegend/HOJA-ESP32-Baseband` observado tem natureza de firmware standalone.

Isso significa que um novo projeto nao deveria simplesmente copiar a pasta e esperar que ela funcione como componente do IDF.

Para virar componente de verdade, ele precisaria de uma adaptacao estrutural.

## Licao 3: o seu hardware futuro deve ser escolhido junto com a camada de leitura

Se voce for para:

- GPIO direto

o caminho de reaproveitamento do `blu-common` fica simples.

Se voce for para:

- matriz de botoes com portas de scan e diodos

sera necessario um componente novo de leitura.

## Licao 4: troca de protocolo por particao e util, mas nao e a unica opcao

O `fw_v2` mostra um caminho funcional:

- cada protocolo ou modo em uma imagem separada
- reboot para trocar

Isso funciona.

Mas para um projeto novo, tambem da para pensar em:

- firmware unico com varios backends de protocolo
- ou firmware unico Switch-first, com expansao posterior

---

## A pergunta central: como criar um novo projeto usando `HOJA-baseband` como componente?

Aqui existe um ponto tecnico importante.

Hoje, o `HandHeldLegend/HOJA-ESP32-Baseband` nao se apresenta como componente IDF reutilizavel pronto. Ele se apresenta como um projeto completo, com `CMakeLists.txt` proprio, `main/` proprio e controle do proprio boot.

Entao, para usá-lo como componente, existem dois caminhos reais.

## Caminho A: usar o `HOJA-baseband` como componente verdadeiro, apos refatoracao

Esse e o caminho tecnicamente mais correto se a exigencia for literalmente "usar o baseband como componente".

### O que precisa ser feito

1. extrair o miolo funcional do baseband para um componente novo
2. remover a dependencia de `app_main()` dentro dele
3. expor uma API publica, por exemplo:
   - `baseband_init()`
   - `baseband_set_mode()`
   - `baseband_submit_input()`
   - `baseband_get_status()`
4. separar a camada I2C slave do nucleo de protocolo
5. deixar a leitura de hardware fora do baseband

### Como o novo projeto ficaria

Camadas sugeridas:

- `components/hardware-input`
- `components/input-matrix` ou `components/input-gpio`
- `components/hoja-baseband-component`
- `main/`

Fluxo:

1. hardware le botoes e sticks
2. camada de input aplica shift e outras regras
3. app envia estado final para o baseband via API
4. baseband gera e transmite o protocolo ativo

### Vantagens do caminho A

- arquitetura limpa
- `HOJA-baseband` vira realmente reutilizavel
- hardware e protocolo ficam desacoplados

### Desvantagens do caminho A

- exige refatoracao consideravel do baseband
- nao e a rota mais curta para um prototipo rapido

## Caminho B: usar o `HOJA-baseband` como referencia forte e criar um wrapper novo

Esse e o caminho mais pragmatico.

Em vez de tentar transformar o baseband inteiro em componente imediatamente, o novo projeto pode:

1. criar um componente novo de protocolo inspirado no baseband
2. portar apenas os blocos necessarios
3. manter hardware, input e politica de modo em componentes separados

### Estrutura sugerida

`components/hardware-input`

- GPIO direto ou matriz
- sticks analogicos ou digitais
- leitura de energia

`components/input-policy`

- shift
- debounce
- traducao do stick digital
- combinacoes especiais

`components/protocol-hoja-baseband`

- codigo reaproveitado do baseband para Switch
- futuramente outros protocolos

`components/mode-manager`

- troca por particao OTA ou por backend interno

`main/`

- somente orquestracao

### Vantagens do caminho B

- mais facil de evoluir
- respeita a separacao de responsabilidades do `fw_v2`
- permite reaproveitar a logica boa de `blu-common`

### Desvantagens do caminho B

- nao e o baseband "puro" como componente literal
- exige disciplina para nao misturar responsabilidades

---

## Minha recomendacao tecnica

Se o objetivo e sair com um projeto novo viavel e bem estruturado, a melhor rota e esta:

### Recomendacao principal

Criar um projeto novo em arquitetura modular, usando o `HOJA-baseband` como origem do componente de protocolo, nao como pasta copiada e plugada diretamente sem refatoracao.

Em termos práticos:

- reaproveitar a ideia de separacao do `fw_v2`
- reaproveitar o comportamento de `shift` do `blu-switch`
- reaproveitar a disciplina de `blu-common` para configuracao de hardware
- extrair do `HOJA-baseband` apenas a camada de protocolo e transporte que interessa

Se fosse para resumir em uma frase:

o projeto novo deve herdar a arquitetura de camadas do `fw_v2`, mas herdar o motor de protocolo do `HOJA-baseband`.

---

## Arquitetura recomendada para o projeto novo

## Camada 1: hardware input

Componente novo.

Sugestao de nome:

- `bebop-input-hw`

Responsabilidades:

- leitura de GPIO direto ou matriz escaneada
- leitura de analogico esquerdo
- leitura de cluster digital complementar
- leitura de botao shift
- leitura de botao de modo ou protocolo

Se o hardware final realmente usar `SCN_PORT_A` a `SCN_PORT_D`, essa camada deve conter um scanner matricial proprio.

## Camada 2: input policy

Componente novo.

Sugestao de nome:

- `bebop-input-policy`

Responsabilidades:

- aplicar regras de shift
- converter stick digital em eixo
- converter stick digital em D-pad quando shift estiver ativo
- consolidar estado final do controle

Essa camada deve produzir uma estrutura limpa de input final.

## Camada 3: protocol core

Componente novo, derivado do `HOJA-baseband`.

Sugestao de nome:

- `bebop-hoja-protocol`

Responsabilidades:

- inicializar Bluetooth
- escolher protocolo ativo
- codificar relatorios
- lidar com pareamento e eventos do host

Esse componente deve nascer do baseband, mas sem o acoplamento de projeto standalone.

## Camada 4: mode manager

Opcional no inicio.

Sugestao de nome:

- `bebop-mode-manager`

Responsabilidades:

- troca entre protocolos por OTA
- ou troca de backend por configuracao interna

No prototipo inicial, da para evitar esse componente e subir apenas o modo Switch.

## Camada 5: app main

Minima.

Responsabilidades:

- inicializar componentes
- costurar callbacks
- rodar loop principal se necessario

---

## Como eu estruturaria o repositorio novo

```text
novo-projeto/
  CMakeLists.txt
  sdkconfig.defaults
  partitions.csv
  components/
    bebop-input-hw/
    bebop-input-policy/
    bebop-hoja-protocol/
    bebop-mode-manager/
  main/
    CMakeLists.txt
    main.c
```

Se quiser manter referencia direta ao codigo original:

```text
components/
  hoja-baseband-vendor/
  bebop-hoja-protocol/
```

onde:

- `hoja-baseband-vendor` guarda o codigo de origem
- `bebop-hoja-protocol` expoe a API integrada ao novo projeto

Isso evita editar agressivamente o vendor logo de cara.

---

## O que reaproveitar de cada projeto

## Do `blu-switch`

- fluxo de boot enxuto
- implementacao de shift minimalista
- ideia de callback para hardware e rumble
- estrategia de troca de imagem via OTA, se voce quiser protocolo por firmware

## Do `blu-common`

- Kconfig de botoes
- padrao de configuracao por GPIO
- leitura de sticks analogicos
- leitura de stick por botoes digitais
- leitura de triggers analogicos

## Do `HOJA-baseband`

- motor de protocolo Switch
- transporte Bluetooth
- resposta a subcomandos
- haptics
- persistencia de MAC e estado relacionado

## Do `HOJA-LIB-ESP32`

- a ideia de frontend com callbacks
- organizacao por cores
- a nocao de desacoplar input da implementacao do protocolo

---

## O que nao reaproveitar sem cuidado

### 1. O `HOJA-baseband` como projeto standalone inteiro

Copiar e chamar isso de componente seria uma falsa modularizacao.

### 2. A suposicao de GPIO direto do `blu-common`

Se o hardware final for matriz escaneada, isso precisa mudar.

### 3. A troca de protocolo apenas por reboot desde o primeiro dia

Isso funciona, mas pode atrasar a consolidacao do protocolo principal.

Minha recomendacao e:

- primeiro estabilizar Switch
- depois decidir se o multi-protocolo sera por backend unico ou por imagens OTA separadas

---

## Proposta de implementacao por fases

## Fase 1: projeto novo, Switch only

Objetivo:

- criar a nova estrutura
- integrar a leitura de hardware
- subir apenas Switch

Entregas:

- novo repo ou nova pasta de firmware
- componente de hardware
- componente de policy
- primeiro wrapper do protocolo Switch vindo do baseband

## Fase 2: decidir GPIO direto versus matriz

Objetivo:

- congelar a topologia real da placa

Se for GPIO direto:

- reaproveitar mais de `blu-common`

Se for matriz com diodos:

- criar `matrix_scanner.c` e `matrix_scanner.h`
- definir linhas, colunas, debounce e ghost prevention

## Fase 3: consolidar shift

Objetivo:

- implementar exatamente o comportamento minimalista ja validado no `fw_v2`

Inclui:

- `A -> X`
- `B -> Y`
- `L -> ZL`
- `R -> ZR`
- `START -> HOME`
- `SELECT -> CAPTURE`
- stick digital virando D-pad no shift

## Fase 4: escolher modelo de multi-protocolo

Opcao A:

- multiplas imagens OTA

Opcao B:

- firmware unico com multiplos backends

Minha recomendacao inicial:

- comecar por Switch only
- depois avaliar `XInput`
- so depois decidir se vale manter multi-imagem ou partir para backend unico

## Fase 5: adicionar protocolo seguinte

Naturalmente, o proximo candidato seria:

- `XInput`, se o core ou branch externa for viavel

ou

- `SInput`, se voce quiser um caminho rapido de bancada

---

## Veredito final

Os dois projetos analisados ensinam bastante, mas nao do jeito que a arvore sugere a primeira vista.

`blu-switch` e `blu-common` mostram um projeto modular com boa separacao entre hardware e framework.

Entao, se a meta e criar um projeto novo usando o `HOJA-baseband` como componente, a conclusao correta e esta:

- nao vale simplesmente copiar a pasta e chamar de componente
- vale extrair dele a camada de protocolo
- vale combinar isso com uma camada de hardware bem separada, no estilo do `fw_v2`
- vale assumir desde o inicio que uma matriz com diodos exigira um scanner proprio

Se fosse para escolher uma frase de arquitetura para o projeto novo, seria esta:

usar o desenho modular do `fw_v2` com o motor de protocolo herdado do `HOJA-baseband`, em vez de tentar encaixar o baseband standalone inteiro dentro da aplicacao.
