# Analise do repositorio fw_v3 e plano de acao para um gamepad multi-protocolos focado em Nintendo Switch

## Contexto adicional do projeto

Os esclarecimentos abaixo refinam o escopo desta analise e corrigem algumas suposicoes iniciais:

- o analogico fisico previsto e o esquerdo
- este repositorio sera usado como componente, nao como firmware final completo do controle
- o objetivo de hardware e transformar uma shell de Game Boy Advance em um controle Bluetooth
- alem dos inputs originais do GBA, serao adicionados botoes especiais estrategicos
- quando o termo `perfil` apareceu na descricao inicial, o significado correto para este projeto e `protocolo`
- o objetivo principal nao e criar varios perfis de mapeamento, e sim maximizar a quantidade de protocolos suportados
- existe a informacao de que o projeto de origem possui uma branch separada para `XInput`, mas essa branch nao esta presente neste clone anexado
- o ambiente de bancada atual usa `ESP32 DevKit1`
- ha interesse futuro em testar `ESP32-S3` bare panel
- ainda existe a intencao de adicionar um display interativo
- espaco interno e custo sao restricoes centrais para a viabilidade do produto

Esses pontos mudam a leitura do documento: a prioridade tecnica deixa de ser "multi-perfis de mapeamento" e passa a ser "multi-protocolo", com foco primario no modo Nintendo Switch.

## Perguntas respondidas

### Compatibilidade de MCU

> O `ESP32C3 Mini` e compativel?

Nao para o caminho principal deste repositorio.

O clone anexado esta configurado para `ESP32` classico, com `CONFIG_IDF_TARGET="esp32"`, `CONFIG_BT_HID_DEVICE_ENABLED=y` e `CONFIG_BTDM_CTRL_MODE_BTDM=y`. O modo Switch implementado aqui depende do caminho de Bluetooth Classic HID. O `ESP32-C3` nao oferece Bluetooth Classic BR/EDR, apenas BLE. Na pratica:

- `ESP32 DevKit1` classico: compativel com a base atual
- `ESP32-C3 Mini`: nao compativel com o modo Switch deste repo
- `ESP32-S3`: tambem nao e substituicao direta para este caminho de Switch via Classic HID

Se a prioridade continuar sendo Nintendo Switch com este firmware como base, o chip seguro e o `ESP32` classico.

> Quais modelos de `ESP32` sao compativeis com o protocolo Switch neste projeto?

Para este firmware, o criterio real de compatibilidade e simples: o chip precisa ter `Bluetooth Classic BR/EDR`, porque o modo Switch implementado aqui usa esse caminho, nao BLE puro.

Na pratica, os modelos compativeis sao os baseados no `ESP32` original, por exemplo:

- `ESP32-WROOM-32`
- `ESP32-WROOM-32D`
- `ESP32-WROOM-32E`
- `ESP32-WROOM-32U` e `32UE`
- `ESP32-WROVER` e variantes equivalentes
- `ESP32-PICO-D4`
- chips bare baseados no `ESP32` original, como variantes `ESP32-D0WD`

Em resumo, a regra segura e esta:

- compativel: familia `ESP32` original com Bluetooth Classic
- nao compativel para este caminho de Switch: `ESP32-S2`, `ESP32-S3`, `ESP32-C2`, `ESP32-C3`, `ESP32-C6`, `ESP32-H2` e outras linhas BLE-only ou sem BR/EDR

Se voce quiser uma frase curta para guiar compra e PCB: para Switch com este repo, procure sempre modulo ou chip baseado no `ESP32` original, nao nas familias `S` ou `C`.

> Qual e a opcao de menor custo possivel e real?

Aqui vale separar tres niveis, porque "mais barato" pode significar coisas diferentes.

#### Menor custo absoluto de BOM

- chip bare `ESP32` original na PCB, com flash externa, cristal, RF e antena feitos por voce

Esse e o menor custo por unidade em volume, mas nao e o menor custo real de projeto no seu caso, porque aumenta bastante:

- risco de RF
- tempo de desenvolvimento
- dificuldade de layout
- chance de retrabalho

#### Menor custo realista para um projeto que precisa funcionar

- um modulo generico baseado em `ESP32-WROOM-32E` ou `ESP32-WROOM-32D`

Essa costuma ser a melhor resposta pratica. O modulo ainda e barato, simplifica RF, acelera prototipo e reduz risco. Para um controle compacto dentro de shell de GBA, isso normalmente e o melhor equilibrio entre custo, tempo e chance real de sucesso.

#### Menor area com risco ainda administravel

- `ESP32-PICO-D4`

Essa opcao pode ser interessante se o problema principal virar espaco interno, porque ela reduz area de placa. Em compensacao, costuma nao ser a opcao mais barata nem a mais simples de comprar.

Conclusao objetiva:

- se voce quer o menor custo possivel no papel: chip bare `ESP32` original
- se voce quer a menor opcao barata que ainda e realista para construir sem sofrer demais: `ESP32-WROOM-32E` ou `ESP32-WROOM-32D`
- se o problema principal for area fisica e nao apenas preco: avaliar `ESP32-PICO-D4`

> Para usar `ATmega` ou `RP2040` direto na PCB, e tranquilo ou vou sofrer?

Os dois sao viaveis, mas o `RP2040` e a opcao mais confortavel se voce seguir com dois MCUs.

`RP2040` oferece:

- mais margem de processamento
- toolchain mais amigavel para firmware proprio
- melhor folga para leitura de botoes, debounce, shift, display e logica auxiliar
- menos risco de engessar o projeto cedo demais

`ATmega` faz sentido apenas se a funcao dele for muito enxuta, por exemplo:

- ler botoes
- aplicar shift simples
- enviar estado final para o ESP32

Se entrar display, menus, configuracao ou mais logica de estado, o `ATmega` tende a apertar rapido.

Resumo pragmatico:

- dois MCUs com baseband Bluetooth separado: `RP2040 + ESP32` e a combinacao mais segura
- `ATmega + ESP32`: possivel, mas com menos margem e mais chance de retrabalho

### MCU unico e multiboot

> Para MCU unico, nao da para usar multiboot com imagem mesclada em flash?

Da para fazer multiboot no sentido de manter mais de uma aplicacao na flash, mas isso nao resolve o problema exatamente como o comando `merge_bin` sugere.

`esptool.py merge_bin` apenas junta varios binarios em um arquivo final com enderecos fixos. Isso nao cria sozinho uma troca de protocolo em runtime. Para haver multiboot real, voce precisaria de uma destas abordagens:

- bootloader customizado que escolhe qual aplicacao iniciar
- esquema OTA com mais de uma particao de app e reboot entre imagens

No `fw_v3` atual, o projeto esta em `single app large`, nao em uma estrutura pronta para alternar varias aplicacoes independentes. E mesmo que voce montasse esse sistema, ainda haveria um limite importante: trocar protocolo por multiboot implica reiniciar o chip e subir outra firmware.

Conclusao pratica:

- multiboot e tecnicamente possivel
- `merge_bin` sozinho nao entrega isso
- isso serve melhor para trocar firmware por reboot do que para dar uma experiencia fluida de multi-protocolo
- se MCU unico for prioridade, o desenho melhor e um firmware unico com camada de input comum e backends de protocolo separados

### Multi-protocolo e branch de XInput

> Quero o maximo de protocolos possivel. O repo tem branch para `XInput`.

Isso muda a prioridade do plano.

Neste clone local, os modos realmente implementados continuam sendo:

- Switch
- SInput

Existe enum para outros modos, inclusive `INPUT_MODE_XINPUT`, mas aqui ele nao tem backend funcional. Se existe uma branch separada para `XInput` no projeto de origem, ela deve ser tratada como uma frente propria de analise, porque nao esta incorporada nesta arvore anexada.

A estrategia correta passa a ser:

1. consolidar `Switch` como protocolo principal
2. manter `SInput` como protocolo secundario util para bancada e testes
3. avaliar a branch `XInput` como codigo separado antes de planejar integracao
4. so depois pensar em consolidar varios protocolos num firmware unico ou numa estrategia de builds por variante

### Display, espaco e custo

> Ainda quero adicionar um display interativo, mas espaco e custo sao criticos.

Esse ponto pesa bastante na arquitetura.

Se voce colocar tudo em um unico MCU, o display concorre com:

- GPIO disponivel
- memoria
- tempo de CPU
- consumo
- complexidade de firmware

Se voce usar dois MCUs, fica mais facil separar responsabilidades, mas aumenta:

- area de placa
- custo de BOM
- complexidade de integracao

Leitura pragmatica para uma shell de GBA:

- se o display for realmente importante, dois MCUs ficam mais defensaveis
- se o objetivo numero um for minimizar espaco, custo e risco, o display deve entrar depois do protocolo principal estar estavel

### Reaproveitamento do SHIFT

> Ja existe um mapeamento de `SHIFT` usado em outro projeto.

Isso e uma vantagem concreta, porque a logica ja esta madura e pode virar regra do projeto.

Pelo trecho trazido, o comportamento desejado ja esta bem definido:

- `A -> X`
- `B -> Y`
- `L -> ZL`
- `R -> ZR`
- `START -> HOME`
- `SELECT -> CAPTURE`
- `ANALOG -> DPAD`

Mesmo que este repositorio fique apenas como componente Bluetooth, essa regra de `SHIFT` deve ser preservada como contrato da camada de input do produto. Em outras palavras:

- em arquitetura de dois MCUs, essa logica vive melhor no MCU que le os botoes
- em arquitetura de MCU unico, essa logica deve existir antes do backend de protocolo

O ponto importante e nao enterrar essa regra dentro de `core_bt_switch.c`. Ela precisa existir como camada propria, porque e reutilizavel para qualquer protocolo que voce decidir adicionar depois.

## Objetivo deste documento

Este documento registra uma analise tecnica do repositorio `fw_v3`, anexado no workspace e descrito como derivado de `HOJA-ESP32-Baseband`, e traduz essa analise em um plano de acao para o seu projeto:

- gamepad minimalista
- foco principal em protocolo Nintendo Switch
- botao tipo `function` ou `shift`
- emulacao do analogico direito usando botoes digitais
- quando combinado com `shift`, esse mesmo conjunto deve se comportar como D-pad normal
- varios protocolos, com foco em Switch primeiro e expansao posterior para outros backends compativeis

O foco aqui nao e apenas descrever arquivos, mas explicar como esse firmware realmente funciona, o que ele ja resolve, o que ele nao resolve e qual e o melhor caminho para usa-lo como base do seu produto.

---

## Resumo executivo

O `fw_v3` nao e um firmware completo de controle no sentido classico. Ele e, na pratica, um firmware de baseband Bluetooth HID para ESP32 que recebe os estados de entrada por I2C e os traduz para modos HID Bluetooth, principalmente:

- Nintendo Switch Pro Controller via Bluetooth Classic HID
- SInput via Bluetooth Classic HID

Isso significa que ele foi desenhado para trabalhar como um processador de comunicacao Bluetooth, enquanto outro bloco do sistema fornece os inputs crus do controle por I2C.

Essa conclusao e a mais importante para o seu projeto.

Se o seu hardware final tiver:

- um microcontrolador principal lendo botoes, shift, selecao de protocolo e logica local
- e um ESP32 dedicado ao papel Bluetooth

entao esse repositorio faz sentido como base. Se voce quer que um unico ESP32 leia diretamente os botoes do controle, aplique shift, selecione protocolo e ainda faca toda a pilha Bluetooth sozinho, este repositorio nao esta pronto para isso e vai precisar de uma camada nova de leitura de GPIO e gerenciamento de estado.

Em outras palavras:

- como firmware baseband Bluetooth, o repositorio e util
- como firmware completo do controle, ele esta incompleto

---

## O que o repositorio e na pratica

### Papel do firmware

O codigo em `main/main.c` mostra um loop principal que:

1. inicializa NVS
2. configura um slave I2C customizado
3. recebe pacotes I2C
4. interpreta comandos
5. despacha os dados recebidos para um backend Bluetooth HID

Os backends reais implementados sao:

- `main/core_bt_switch.c`
- `main/core_bt_sinput.c`

Isso deixa claro que a funcao central do firmware e converter uma estrutura de input recebida externamente em relatórios HID Bluetooth.

### Fluxo de dados

O fluxo real hoje e este:

1. outro processador ou firmware gera um pacote de input
2. esse pacote chega ao ESP32 por I2C slave no endereco `0x76`
3. `main.c` valida CRC e converte o pacote para `i2cinput_input_s`
4. um callback de modo ativo traduz isso para o formato do protocolo escolhido
5. uma task Bluetooth manda periodicamente os relatórios HID

Fluxo resumido:

`host externo -> I2C -> fw_v3 -> mapeamento HID -> Bluetooth`

---

## Ambiente tecnico identificado

Pelos arquivos de build e configuracao, o ambiente real observado neste clone e:

- alvo: `esp32`
- arquitetura: `xtensa`
- ESP-IDF: `v5.5.0`
- flash configurada: `2MB`
- tabela de particao: `single app large`
- Bluetooth dual mode habilitado no sdkconfig
- Bluetooth Classic HID habilitado
- BLE habilitado
- SDP habilitado

Arquivos que sustentam isso:

- `sdkconfig`
- `build/project_description.json`
- `build/config.env`

Apesar do BLE estar habilitado no ambiente, os modos efetivamente usados pelo firmware anexado estao orientados ao caminho Classic HID.

---

## Estrutura principal do repositorio

### Arquivos centrais

#### `main/main.c`

Arquivo central do sistema.

Responsabilidades:

- inicializacao de NVS
- configuracao do I2C slave
- validacao CRC dos pacotes
- armazenamento de MACs emparelhados
- roteamento dos comandos I2C
- escolha do modo Bluetooth
- coleta de bateria por ADC interno
- enfileiramento de mensagens de retorno, como haptics e status

#### `main/include/hoja_includes.h`

Arquivo mais importante para entender o contrato de input.

Ele define:

- os modos de entrada (`INPUT_MODE_*`)
- a estrutura `i2cinput_input_s`
- includes centrais do projeto

Essa estrutura de input e o ponto de entrada de toda a logica funcional do controle.

#### `main/include/hoja_types.h`

Define tipos persistidos e tipos auxiliares importantes:

- `hoja_settings_s`
- `hoja_live_s`
- status de bateria
- status I2C de retorno
- estruturas de IMU e haptics

#### `main/core_bt_switch.c`

Implementa o modo Nintendo Switch Pro Controller.

Responsabilidades:

- descritor HID de Pro Controller
- callbacks de GAP e HID
- logica de pareamento e reconexao
- task de envio continuo de relatorios
- traducao direta de `i2cinput_input_s` para relatorio Switch

#### `main/core_bt_sinput.c`

Implementa um modo SInput, com suporte a recursos como:

- motion
- haptics
- feature flags por produto
- triggers analogicos

Serve como segundo backend funcional do projeto.

#### `main/switch_commands.c`

Implementa o lado do protocolo Switch que responde a subcomandos do host Nintendo Switch, como:

- info do dispositivo
- player leds
- IMU enable
- leitura de SPI virtual
- vibration enable

#### `main/switch_analog.c`

Implementa codificacao e calibracao de valores analogicos para o formato do Switch.

#### `main/util_bt_hid.c`

Camada utilitaria para inicializacao da pilha Bluetooth e registro do dispositivo HID.

#### `main/mitch_i2c.c`

Driver I2C customizado. E parte critica da arquitetura porque o firmware depende totalmente dele para receber os inputs.

#### `main/imu_tool.c`

Armazena e compacta dados de IMU, inclusive para formatos usados pelo Switch.

---

## Modos suportados hoje

O enum de modos em `hoja_includes.h` declara:

- `INPUT_MODE_SWPRO`
- `INPUT_MODE_XINPUT`
- `INPUT_MODE_GCUSB`
- `INPUT_MODE_GAMECUBE`
- `INPUT_MODE_N64`
- `INPUT_MODE_SNES`
- `INPUT_MODE_SINPUT`

Mas, no codigo real anexado, os modos implementados de fato sao apenas:

- `INPUT_MODE_SWPRO`
- `INPUT_MODE_SINPUT`

Os demais aparecem como declaracoes ou heranca conceitual do projeto original, mas nao possuem backend funcional neste `fw_v3`.

Conclusao pratica:

- o repositorio nao entrega hoje "o maximo de perfis possiveis" no nivel de protocolo Bluetooth
- ele entrega apenas dois backends reais
- a prioridade natural para o seu projeto deve ser Switch primeiro, SInput como bonus, e expansao futura so depois de consolidar a arquitetura interna de perfis

---

## Como o firmware recebe os inputs

### Estrutura de input atual

A estrutura principal de entrada e `i2cinput_input_s`.

Ela contem:

- D-pad digital
- face buttons
- ombros e gatilhos digitais
- plus e minus
- clique dos analogicos
- botoes de sistema como home, capture, sync, unbind
- `lx`, `ly`, `rx`, `ry`
- `lt`, `rt`
- acelerometro
- giroscopio
- estado de energia

Ou seja, o contrato atual ja pressupoe que algum outro bloco do sistema transformou leituras fisicas de botoes e sensores nesse formato.

### Consequencia importante para o seu projeto

O seu conceito de:

- botao `function`
- combinacoes para perfis
- reinterpretacao do cluster do analogico direito
- shift transformando funcao do mesmo conjunto de botoes

nao existe hoje nesse firmware.

Hoje ele apenas recebe um estado de input pronto e repassa para o backend do modo ativo.

---

## Como o modo Switch funciona

O modo Switch em `core_bt_switch.c` imita um Pro Controller classico.

### O que ja esta pronto

- descritor HID de Pro Controller
- nome do dispositivo `Pro Controller`
- fabricante `Nintendo`
- vendor/product ID compatíveis com esse modo
- resposta a subcomandos esperados pelo Switch
- envio periodico de relatorios de 48 bytes
- suporte a IMU
- suporte a vibration por relatorios de saida
- armazenamento de MAC do host emparelhado

### Como o mapeamento e feito hoje

O mapeamento e direto, fixo e sem camada intermediaria.

Exemplos da ideia atual:

- `button_east -> A`
- `button_south -> B`
- `button_north -> X`
- `button_west -> Y`
- `dpad_* -> dpad do Switch`
- `lx/ly/rx/ry -> sticks`

Isso e simples e funcional, mas cria um problema estrutural para o seu objetivo: nao existe um sistema de remapeamento por perfil.

---

## Como o modo SInput funciona

O modo SInput em `core_bt_sinput.c` e mais flexivel em alguns pontos.

Ele oferece:

- um report descriptor proprio
- estruturas maiores de input
- haptic command channel
- feature flags variando por produto e sub-ID
- envio de motion e triggers em formato generico

Para o seu produto, o SInput pode ser util como perfil secundario para PCs, ferramentas internas ou testes, mas o valor principal desta base esta no modo Switch.

---

## Persistencia atual em NVS

O projeto grava em NVS sob o namespace `hsettings`.

Hoje, o que realmente e persistido e principalmente:

- MAC do dispositivo Switch
- MAC do dispositivo SInput
- MAC do host pareado para Switch
- MAC do host pareado para SInput
- magic number de validacao

Isso significa que:

- nao existe persistencia de perfis
- nao existe persistencia de remapeamentos
- nao existe persistencia de camadas de shift
- nao existe persistencia de configuracao do cluster digital do analogico direito

Para o seu projeto, isso precisa mudar logo no inicio da arquitetura.

---

## Pontos fortes do repositorio para o seu objetivo

### 1. Base Switch ja funcional

Esse e o maior valor do repositorio. O trabalho dificil de fazer um ESP32 se comportar como controle Bluetooth para Nintendo Switch ja esta bastante encaminhado.

### 2. Contrato de input ja definido

Mesmo que ainda falte a logica de perfis, ja existe um formato claro de dados de entrada.

### 3. Haptics e IMU ja tem fundacao

Se o seu hardware final suportar isso, o caminho esta parcialmente aberto.

### 4. Separacao entre input bruto e backend de protocolo

Mesmo sem uma camada formal de remapeamento, a arquitetura ja separa:

- recepcao do input
- envio Bluetooth

Isso favorece a insercao de uma camada nova entre as duas.

### 5. Modo SInput como caminho secundario

Permite manter um perfil alternativo generico sem precisar começar do zero.

---

## Limitacoes importantes do repositorio

### 1. Nao e um firmware de controle completo

Esse e o maior limite. O projeto espera receber input por I2C. Ele nao escaneia botoes do controle por GPIO no estado atual.

### 2. Nao existe sistema de perfis

Nao ha:

- tabela de remapeamento
- perfil ativo
- combinacao para troca de perfis
- persistencia de perfis
- API para configuracao de perfis

### 3. Nao existe botao `function` dedicado na logica

Embora existam bits de botoes de sistema, nenhum representa hoje o comportamento de camada funcional que voce descreveu.

### 4. Mapeamento atual e direto demais

Os backends convertem `i2cinput_input_s` diretamente para relatorios HID. Isso obriga a inserir uma camada nova se quisermos suportar:

- shift
- perfis
- macros simples
- reinterpretacao contextual dos botoes

### 5. Modos declarados nao significam modos implementados

O enum sugere muitos perfis de console, mas o clone anexado nao entrega isso de verdade.

### 6. Mudanca de protocolo nao e hot-swap simples

O firmware seleciona um backend no startup do modo Bluetooth. Alternar Switch para SInput ou outro protocolo envolve reinicializacao da pilha Bluetooth, nao apenas troca instantanea de mapeamento.

### 7. Estado compartilhado pouco formalizado

Alguns dados globais sao atualizados de forma direta, e isso merece cuidado ao adicionar perfis dinâmicos e troca em tempo real.

---

## Conclusao tecnica sobre adequacao ao seu projeto

### O repositorio serve?

Sim, mas com uma condicao clara.

Ele serve muito bem como:

- base de comunicacao Bluetooth Switch
- backend HID para um projeto com arquitetura em camadas

Ele nao serve, no estado atual, como implementacao final do comportamento do seu controle minimalista.

### O ponto certo para adaptar

O melhor lugar para adaptar esse repo nao e espalhar `if` dentro de `core_bt_switch.c` e `core_bt_sinput.c`.

O caminho correto e inserir uma nova camada de transformacao entre:

- input recebido
- backend de protocolo

Essa camada deve ser responsavel por:

- perfil ativo
- botao `function`
- combinacoes de perfil
- mapeamento contextual
- emulacao do analogico direito por botoes digitais
- modo alternativo do mesmo cluster quando `shift` estiver pressionado

---

## Arquitetura recomendada para o seu projeto

## Visao geral

Recomendo organizar o projeto em 4 camadas logicas.

### Camada 1: input fisico

Responsavel por ler:

- botoes reais
- cluster do analogico direito digital
- dpad fisico, se existir
- botao `function`
- botao de perfil
- sensores opcionais

Se houver outro MCU no sistema, essa camada pode viver fora do `fw_v3`.

Se o ESP32 for unico, essa camada precisa ser criada dentro do firmware.

### Camada 2: interpretacao logica

Nova camada a ser criada.

Responsabilidades:

- debouncing
- reconhecimento de combinacoes
- troca de perfil
- aplicacao da camada `function`
- conversao do cluster digital para `rx/ry`
- conversao alternativa do mesmo cluster para D-pad quando `shift` estiver ativo
- producao de um `i2cinput_input_s` final e consistente

### Camada 3: adaptacao por protocolo

Ja existe em grande parte.

Responsabilidades:

- `switch_bt_sendinput`
- `sinput_bt_sendinput`

Essa camada nao deveria conhecer botao de perfil nem logica de shift. Ela deve receber um input ja resolvido.

### Camada 4: transporte Bluetooth

Ja existe em grande parte.

Responsabilidades:

- pareamento
- reconexao
- envio de relatorios
- resposta a subcomandos Switch

---

## Melhor estrategia para o seu caso especifico

### Decisao arquitetural principal

Voce precisa escolher entre dois caminhos.

### Caminho A: manter o conceito baseband

Um MCU principal faz:

- leitura de botoes
- logica de perfis
- shift
- combinacoes
- definicao do estado final do input

O `fw_v3` faz apenas:

- Bluetooth
- protocolo Switch/SInput

Vantagens:

- menos mudancas no repo
- preserva a arquitetura original
- menor risco sobre a parte Bluetooth

Desvantagens:

- precisa de dois blocos de firmware ou dois processadores

### Caminho B: transformar o ESP32 em firmware completo do controle

O proprio `fw_v3` passa a fazer:

- leitura de GPIO
- perfis
- layers
- logica de funcao
- backend Bluetooth

Vantagens:

- produto final mais integrado
- menos dependencia de um host de input externo

Desvantagens:

- exige bem mais trabalho
- foge mais do desenho original do repo
- aumenta risco de regressao em Switch

### Recomendacao

Para chegar mais rapido a um prototipo funcional, recomendo:

- primeiro seguir o Caminho A para validar experiencia de uso e mapeamento
- depois, se fizer sentido para o hardware final, migrar para o Caminho B

---

## Plano de acao proposto

## Fase 0: definicao funcional do controle

Objetivo: congelar a logica do seu produto antes de alterar o firmware.

Entregas:

- lista exata de botoes fisicos
- definicao do botao `function`
- definicao do botao dedicado de perfil
- definicao do cluster que vai emular o analogico direito
- tabela de combinacoes para troca de perfis
- tabela de acoes por perfil

Documento que precisa sair desta fase:

- matriz fisico -> logico

Exemplo do que precisa ser decidido:

- sem shift, cluster direito = `rx/ry`
- com shift, cluster direito = D-pad
- `profile + A` = perfil 1
- `profile + B` = perfil 2
- `profile + X` = perfil 3
- `profile + Y` = perfil 4

Sem isso, qualquer implementacao vai virar regra espalhada e dificil de manter.

---

## Fase 1: criar uma camada de perfil no firmware

Objetivo: inserir uma camada formal de transformacao entre input bruto e backend HID.

Mudancas recomendadas:

- criar `main/profile_mapper.c`
- criar `main/include/profile_mapper.h`
- definir estruturas de perfil em `main/include/hoja_types.h`

Essa nova camada deve receber um input bruto e devolver um input final ja transformado.

Responsabilidades dessa camada:

- armazenar `active_profile`
- aplicar remapeamentos
- aplicar comportamento do botao `function`
- resolver emulacao de analogico direito por botoes digitais
- resolver troca de perfis

Saida esperada:

- `switch_bt_sendinput()` e `sinput_bt_sendinput()` passam a receber dados ja prontos

---

## Fase 2: modelar perfis persistentes

Objetivo: permitir varios perfis reais e mantidos em flash.

Mudancas recomendadas:

- expandir `hoja_settings_s`
- incluir array de perfis
- incluir `last_active_profile`
- salvar configuracoes no mesmo blob de NVS

Cada perfil deve conter ao menos:

- nome ou ID
- mapa de botoes
- configuracao do cluster direito digital
- comportamento do shift
- se o perfil usa layout principal Switch ou outro backend

Minha recomendacao inicial:

- 4 perfis reais primeiro
- depois expandir para 8 se houver espaco e necessidade

Quatro perfis ja cobrem bem:

- perfil Switch padrao
- perfil Switch alternativo
- perfil SInput/PC
- perfil experimental

---

## Fase 3: implementar o botao `function`

Objetivo: criar uma camada secundaria de mapeamento contextual.

Esse botao nao deve ser implementado dentro do backend Switch.

Ele deve ser resolvido antes.

Comportamentos recomendados:

- enquanto `function` estiver pressionado, uma segunda tabela de mapeamento entra em vigor
- se `function` nao estiver pressionado, usa-se a tabela primaria do perfil

Isso permite que poucos botoes fisicos cubram mais funcoes sem contaminar o backend de protocolo.

---

## Fase 4: emular o analogico direito com botoes digitais

Objetivo: transformar um conjunto de botoes em `rx/ry`.

Implementacao recomendada:

- definir 4 direcoes digitais do cluster direito
- converter combinacoes em valores fixos de `rx` e `ry`
- usar centro em repouso
- usar diagonais quando duas direcoes forem pressionadas juntas

Tabela sugerida:

- repouso = `rx=2048`, `ry=2048`
- direita = maximo no eixo X
- esquerda = minimo no eixo X
- cima = maximo ou minimo no eixo Y conforme convencao final do backend
- baixo = inverso do acima
- diagonais = combinacoes X/Y

Idealmente, essa conversao deve permitir:

- intensidade fixa para versao 1
- opcionalmente curva ou turbo depois

---

## Fase 5: fazer o shift do cluster direito virar D-pad

Objetivo: reutilizar os mesmos botoes para dois papeis diferentes.

Regra funcional desejada:

- normal: cluster direito = analogico direito emulado
- com `function`: cluster direito = D-pad normal

Implementacao correta:

- a camada de perfil zera `rx/ry` para centro quando `function` estiver ativo nesse contexto
- a camada injeta `dpad_up/down/left/right`

Ou seja, o backend Switch continua recebendo um `i2cinput_input_s` coerente, sem precisar saber que o input veio de botoes digitais.

---

## Fase 6: implementar troca de perfis por combinacao

Objetivo: permitir varios perfis sem interface grafica.

Modelo recomendado:

- botao `profile` dedicado
- combinacao com face buttons ou direcoes
- debounce e lockout curto para evitar troca acidental
- confirmacao por haptics, LED ou padrao de status

Exemplo simples:

- `profile + south` = perfil 1
- `profile + east` = perfil 2
- `profile + west` = perfil 3
- `profile + north` = perfil 4

Recomendacoes:

- troca de perfil deve ocorrer na solta ou apos janela curta de confirmacao
- nao misturar troca de perfil com comandos de pareamento
- nao reutilizar `button_sync` para isso se ele ainda for necessario no fluxo Bluetooth

---

## Fase 7: definir a politica de protocolos por perfil

Objetivo: separar dois conceitos que costumam ser confundidos.

Existem dois tipos de perfil:

- perfil de mapeamento interno
- perfil de protocolo Bluetooth

Recomendacao:

- no inicio, todos os perfis internos usam o backend Switch
- SInput fica como perfil especial separado, se necessario

Motivo:

trocar protocolo Bluetooth e muito mais pesado do que trocar mapeamento interno. Primeiro vale consolidar varios perfis dentro do modo Switch.

---

## Fase 8: endurecer a confiabilidade

Objetivo: evitar regressao quando a logica de perfil ficar mais complexa.

Itens recomendados:

- snapshot ou estrategia segura para compartilhamento entre tasks
- validacao de perfil ativo
- fallback para perfil padrao se NVS vier corrompida
- logs de debug para troca de perfil
- validacao de combinacoes impossiveis

---

## Estruturas novas recomendadas

## Perfil

Uma estrutura inicial razoavel seria algo como:

```c
typedef struct {
    uint8_t profile_id;
    uint8_t enabled;
    uint8_t protocol_mode;
    uint8_t function_mode;
    uint16_t primary_button_map[24];
    uint16_t function_button_map[24];
    uint16_t digital_rx_positive;
    uint16_t digital_rx_negative;
    uint16_t digital_ry_positive;
    uint16_t digital_ry_negative;
    uint8_t right_cluster_behavior_default;
    uint8_t right_cluster_behavior_shifted;
} hoja_profile_s;
```

Nao precisa ser exatamente isso, mas a ideia e correta: perfil precisa existir como dado, nao como varios `if` espalhados.

## Estado de runtime

Tambem recomendo um estado de runtime separado:

```c
typedef struct {
    uint8_t active_profile;
    bool function_pressed;
    bool profile_pressed;
    uint32_t last_profile_change_ms;
} hoja_runtime_state_s;
```

---

## O que eu faria primeiro, na pratica

Se eu fosse transformar este repo no seu produto, eu seguiria esta ordem:

1. congelar o layout fisico de botoes
2. definir tabela de funcoes por perfil
3. criar a camada `profile_mapper`
4. implementar apenas um backend final: Switch
5. validar o analogico direito digital
6. validar o comportamento com `function`
7. implementar troca de 4 perfis
8. persistir perfis em NVS
9. so depois considerar SInput como perfil alternativo

Esse encadeamento reduz risco porque ataca primeiro o que mais importa para voce: experiencia no Switch.

---

## O que eu nao recomendo no inicio

- tentar ativar todos os modos do enum como se ja existissem
- colocar logica de perfil dentro de `core_bt_switch.c`
- misturar troca de perfil com logica de pareamento
- reusar indiscriminadamente `button_sync` como `function`
- tentar implementar hot-swap de protocolo Bluetooth logo de cara

Esses caminhos aumentam complexidade e podem quebrar exatamente a parte mais valiosa do repo, que e o comportamento Switch.

---

## Riscos principais para o seu projeto

### 1. Confundir perfil de mapeamento com perfil de protocolo

Se tudo for tratado como "perfil", o desenho fica confuso. E melhor separar:

- perfil interno de botoes
- modo de transporte/protocolo

### 2. Colocar muita logica no backend Switch

Isso tornaria o codigo dificil de expandir para SInput ou outros modos.

### 3. Implementar shift sem uma tabela declarativa

Sem uma tabela central de mapeamento, a manutencao vira um problema rapidamente.

### 4. Depender do I2C sem decidir a arquitetura de hardware

Voce precisa decidir cedo se o ESP32 vai continuar sendo baseband ou se vai virar o MCU principal do controle.

### 5. Troca de perfil em tempo errado

Trocar perfil no meio de uma sequencia de inputs ou pareamento pode causar comportamento estranho. Precisa haver politica clara de debounce e confirmacao.

---

## Avaliacao final

### O que este repo ja resolve bem

- emulacao Bluetooth para Nintendo Switch
- estrutura de envio HID
- parte do emparelhamento e persistencia de MAC
- motion/haptics em fundacao utilizavel
- backend secundario SInput

### O que falta para virar o seu produto

- sistema de perfis
- camada de remapeamento
- botao `function`
- emulacao do analogico direito com botoes digitais
- reinterpretacao do mesmo cluster como D-pad sob shift
- comando ou logica para trocar perfil
- persistencia desses perfis
- eventualmente leitura de GPIO local, caso o ESP32 seja o unico MCU

### Veredito

`fw_v3` e uma boa base tecnica para o seu projeto se usado como nucleo Bluetooth Switch-first, mas ele ainda nao contem a logica de produto que diferencia o seu controle. Essa logica precisara ser implementada em uma camada nova, idealmente antes dos backends de protocolo.

---

## Proximo passo recomendado

O proximo passo mais racional e produzir uma especificacao fechada do layout de botoes e dos perfis desejados, porque isso determina a modelagem da nova camada de mapeamento.

Em seguida, a primeira implementacao concreta deveria ser:

1. criar a camada `profile_mapper`
2. suportar 1 perfil Switch funcional
3. adicionar `function`
4. fazer o cluster direito digital gerar `rx/ry`
5. fazer o mesmo cluster virar D-pad com shift
6. adicionar troca entre 4 perfis

Quando isso estiver estavel, o repositorio passa a ser nao apenas uma baseband Bluetooth, mas o nucleo funcional do seu controle multi-perfis.
