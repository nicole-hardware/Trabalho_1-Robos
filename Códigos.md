**1. Análise do Script 1 de Inicialização do Robô NAO no CoppeliaSim
(responsável por configurar cores e preparar o robô para a simulação)**

O primeiro script do projeto foi desenvolvido em LUA e tem como objetivo principal modificar automaticamente as cores das partes visíveis do robô humanoide NAO dentro do ambiente de simulação CoppeliaSim. Essa mudança é realizada assim que a simulação é iniciada, garantindo uma aparência padronizada e fiel ao modelo real do robô, o que facilita a visualização e a identificação de cada um de seus componentes durante os testes.

Logo no início do código, o comando sim=require'sim' é responsável por importar a biblioteca principal do CoppeliaSim. Essa biblioteca fornece todas as funções necessárias para manipular objetos, propriedades e elementos visuais da cena. Em seguida, são definidas duas variáveis que armazenam as cores que serão aplicadas no robô: w={0.95,0.95,0.95} e g={1,0.5,0.08}. A primeira representa um tom de branco levemente suavizado, e a segunda define um tom alaranjado. Essas cores correspondem às características visuais típicas do robô NAO, que combina o branco predominante com detalhes coloridos.

Depois disso, o script cria duas listas: allObjectsToExplore e allVisibleShapes. A primeira é inicializada com o comando sim.getObject('..'), que obtém o objeto pai do script ,ou seja, o modelo principal do robô NAO. Já a segunda lista (allVisibleShapes) é inicialmente vazia e servirá para armazenar todos os objetos tridimensionais (shapes) que o código irá encontrar e modificar.

Em seguida, o programa entra em um laço de repetição que percorre todos os objetos pertencentes ao modelo do robô. Esse processo é importante porque o NAO é composto por várias partes, cabeça, tronco, braços, pernas, entre outras, e cada uma delas é tratada separadamente dentro da simulação. Assim, o laço garante que nenhuma parte fique de fora da análise.

Dentro desse laço, o comando sim.getObjectType(obj) verifica se o objeto em questão é um shape (isto é, um elemento visível da cena). Caso seja, o script consulta suas propriedades com sim.getIntProperty(obj, 'layer'), que retorna o número da camada em que o objeto está localizado. No CoppeliaSim, as camadas (layers) determinam a visibilidade de cada objeto, apenas as camadas iniciais (valores abaixo de 256) são exibidas por padrão. Dessa forma, o código filtra e seleciona apenas os elementos visíveis do robô, ignorando os que estão ocultos ou desativados.

Além disso, o script verifica também os objetos filhos de cada componente, utilizando a função sim.getObjectChild(obj, index). Essa busca é feita dentro de um laço que continua até não haver mais filhos (quando o valor retornado é -1). Esse processo de varredura garante que todas as partes do robô inclusive as menores ou hierarquicamente internas sejam incluídas na lista de objetos a serem modificados. Assim, a mudança de cor é aplicada de forma completa e uniforme em todo o modelo.

Por fim, o código entra na etapa de aplicação das cores. A função sim.setShapeColor() é utilizada para alterar as propriedades visuais dos shapes identificados. Essa função recebe quatro parâmetros: o objeto, o nome do material (como “NAO_WHITE” ou “NAO_GREY”), o tipo de componente (no caso, 0, que representa a superfície principal) e o vetor RGB da cor desejada. Com isso, o script aplica tanto o tom branco quanto o alaranjado em diferentes partes do robô, reproduzindo o padrão visual característico do NAO físico.

Em síntese, esse script tem um papel fundamental na personalização e identificação visual do robô NAO dentro do CoppeliaSim. Ele automatiza a busca e a modificação das cores de cada componente, tornando o modelo mais realista, organizado e de fácil visualização durante as simulações.

Do ponto de vista acadêmico, o código demonstra de forma prática como a linguagem LUA pode ser usada para manipular objetos tridimensionais em ambientes de simulação robótica. Além disso, exemplifica conceitos essenciais, como hierarquia de objetos, camadas de visibilidade e alteração de propriedades visuais, mostrando a importância de scripts automatizados para a configuração eficiente de modelos robóticos complexos em ambientes virtuais.

**Código 1 – Script de Inicialização do Robô NAO**

```
sim=require'sim'

function sysCall_init() 
    -- Determin the colors we want
    w={0.95,0.95,0.95}
    g={1,0.5,0.08}
    -- Get all the visible shapes in this model:
    allObjectsToExplore={sim.getObject('..')}
    allVisibleShapes={}
    while (#allObjectsToExplore>0) do
        obj=allObjectsToExplore[1]
        table.remove(allObjectsToExplore,1)
        if (sim.getObjectType(obj)==sim.sceneobject_shape) then
            v=sim.getIntProperty(obj, 'layer') -- get the layers this shape is visible in
            if (v<256) then -- by default, the first 8 layers are visible, the last ones are invisible
                table.insert(allVisibleShapes,obj)
            end
        end
        index=0
        while true do
            child=sim.getObjectChild(obj,index)
            if (child==-1) then
                break
            end
            table.insert(allObjectsToExplore,child)
            index=index+1
        end
    end
    -- Now change the color of all those shapes:
    for i=1,#allVisibleShapes,1 do
        sim.setShapeColor(allVisibleShapes[i],'NAO_WHITE',0,w)
        sim.setShapeColor(allVisibleShapes[i],'NAO_GREY',0,g)
    end
end
```


**2. Análise do Script 2 de Detecção de Cores e Controle de Movimento do Robô NAO no CoppeliaSim
(responsável por identificar objetos coloridos, interpretar o ambiente e controlar movimentos dos braços e da cabeça)**

O segundo script do projeto também foi desenvolvido em LUA e tem como principal objetivo fazer com que o robô humanoide NAO consiga identificar cubos coloridos no ambiente de simulação e reagir a eles de forma adequada, como desviar de obstáculos ou levantar os braços em comemoração. Para isso, o código utiliza a detecção de “blobs”, que são manchas de cor captadas pelo sensor de visão do robô. Essas ações acontecem continuamente durante a simulação, permitindo que o NAO responda dinamicamente ao que “vê” no ambiente virtual.

Logo no início, a função sysCall_init() é chamada automaticamente quando a simulação começa. Nela, o script pega os “handles”, que são referências para os objetos dentro do CoppeliaSim, incluindo o próprio robô (nao), a câmera de visão (camera) e os motores dos braços (lShoulderPitch, rShoulderPitch, lShoulderRoll, rShoulderRoll) e da cabeça (headYaw, headPitch). Esses handles são essenciais porque permitem que o código controle cada articulação e acesse as informações visuais do ambiente.

Em seguida, o script define a posição inicial dos braços do NAO, mantendo-os abaixados através da função auxiliar abaixarBracos(). Também é exibida uma mensagem na barra de status avisando que o robô está pronto para detectar cores, dando um retorno visual ao usuário sobre o estado do sistema.

A função sysCall_actuation() é chamada a cada passo da simulação e funciona como o “cérebro” do robô. Nela, o script tenta detectar blobs usando sim.getVisionSensorBlobInfo(camera), que retorna a quantidade de manchas detectadas e informações sobre cada uma delas, como tamanho e valores de cor RGB normalizados entre 0 e 1. O código então analisa o maior blob, assumindo que ele seja o objeto mais relevante naquele momento.

Para descobrir a cor do blob, o script utiliza a função auxiliar identificarCor(r, g, b). Ela compara os valores de vermelho, verde e azul e retorna o nome da cor detectada, considerando cores básicas como preto, vermelho, verde, azul, amarelo e laranja, ou “indefinido” caso os valores não correspondam a nenhuma cor conhecida.

Depois de identificar a cor, o NAO decide como reagir:

Se a cor for verde e ocupar mais de 70% da tela, o robô interpreta que está diante do “jardim” ou de um obstáculo grande, então desvia movendo a cabeça em um gesto de “não” e mantém os braços abaixados.

Se for uma cor específica que não seja o jardim, o NAO “comemora” levantando os braços com a função levantarBracos() e centraliza a cabeça.

Se nenhuma cor for detectada, o robô mantém os braços abaixados e a cabeça neutra, mostrando que ainda está procurando.

As funções auxiliares levantarBracos() e abaixarBracos() controlam com precisão os motores dos braços, definindo ângulos específicos para cada articulação e permitindo movimentos suaves e naturais durante a simulação.

Em resumo, esse script é fundamental para tornar o robô NAO interativo e responsivo dentro do CoppeliaSim. Ele integra percepção visual com controle de movimento, mostrando na prática como sensores e atuadores podem ser programados para reagir a estímulos do ambiente.

Do ponto de vista acadêmico, o código demonstra o uso da linguagem LUA para controlar robôs humanoides em um ambiente virtual. Ele também exemplifica conceitos importantes de robótica, como processamento de informações sensoriais, identificação de cores, controle de motores e tomada de decisões condicionais, mostrando como a programação pode tornar um modelo robótico mais autônomo, dinâmico e capaz de interagir com o ambiente.




**Código 2 – Script de Detecção de Cores e Controle de Movimento do Robô NAO**
```
sim=require'sim'

function sysCall_init() 
    -- Determin the colors we want
    w={0.95,0.95,0.95}
    g={1,0.5,0.08}
    -- Get all the visible shapes in this model:
    allObjectsToExplore={sim.getObject('..')}
    allVisibleShapes={}
    while (#allObjectsToExplore>0) do
        obj=allObjectsToExplore[1]
        table.remove(allObjectsToExplore,1)
        if (sim.getObjectType(obj)==sim.sceneobject_shape) then
            v=sim.getIntProperty(obj, 'layer') -- get the layers this shape is visible in
            if (v<256) then -- by default, the first 8 layers are visible, the last ones are invisible
                table.insert(allVisibleShapes,obj)
            end
        end
        index=0
        while true do
            child=sim.getObjectChild(obj,index)
            if (child==-1) then
                break
            end
            table.insert(allObjectsToExplore,child)
            index=index+1
        end
    end
    -- Now change the color of all those shapes:
    for i=1,#allVisibleShapes,1 do
        sim.setShapeColor(allVisibleShapes[i],'NAO_WHITE',0,w)
        sim.setShapeColor(allVisibleShapes[i],'NAO_GREY',0,g)
    end
end
```
