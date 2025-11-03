## Arquivo do artigo 
[Robôs, sensores e aplicações.pdf](https://github.com/user-attachments/files/23319734/Robos.sensores.e.aplicacoes.pdf)

## Artigo em Latex
```
\documentclass[a4paper, 12pt]{article}

% --- PACOTES ---
\usepackage[utf8]{inputenc}
\usepackage[T1]{fontenc}
\usepackage[brazil]{babel}
\usepackage{geometry}
\usepackage[hidelinks]{hyperref}
\usepackage{textcomp} % Para usar \texttt

% Configuração da página
\geometry{
    a4paper,
    left=25mm,
    right=25mm,
    top=30mm,
    bottom=30mm,
}

% --- INFORMAÇÕES DO DOCUMENTO ---
\title{Trabalho 1: Navegação e Interação do Robô NAO no CoppeliaSim }


\author{
    Aline Maria de Miranda Pereira \\
    Ellen Stefany da Silva \\
    Josiane de Fátima Bueno \\
    Nicole Cristina dos Santos \\[1em] % Adiciona um espaço vertical
    \large Pontifícia Universidade Católica de Minas Gerais
}
\date{\today}

% --- INÍCIO DO DOCUMENTO ---
\begin{document}

\maketitle

% --- RESUMO (ABSTRACT) ---
\begin{abstract}
Este documento detalha a solução técnica implementada para o projeto de controle do robô humanoide NAO no ambiente de simulação CoppeliaSim. O projeto foi desenvolvido para a disciplina de Robôs, Sensores e Aplicações, com foco na programação de percepção e interação. Utilizando a linguagem de script Lua nativa da plataforma, foi desenvolvido um controlador que permite ao NAO identificar visualmente objetos (cubos coloridos) e barreiras (as paredes de um "jardim"). O robô é programado para reagir de forma distinta a cada estímulo: ao identificar um cubo-alvo, ele levanta os braços em sinal de reconhecimento; ao detectar a proximidade de uma parede (identificada pela cor verde e grande tamanho no campo de visão), ele executa um movimento de desvio com a cabeça.
\end{abstract}

\vspace{1cm}
\noindent\textbf{Palavras-chave:} Robô NAO, CoppeliaSim, Script Lua, Percepção Visual, Detecção de Cor, Controle de Robô.

\newpage

% --- SEÇÕES DO ARTIGO ---

\section{Introdução}

\paragraph{} A simulação robótica é uma ferramenta indispensável no desenvolvimento de sistemas autônomos complexos. Plataformas como o CoppeliaSim (anteriormente V-REP) permitem a rápida prototipagem e o teste de algoritmos de controle, percepção e navegação antes de sua implementação em hardware real, como o robô humanoide NAO \cite{coppelia}.

O objetivo deste trabalho é implementar um sistema de controle reativo para o robô NAO. O robô deve operar em um ambiente delimitado (arena "jardim") e ser capaz de, autonomamente, diferenciar objetos-alvo (cubos coloridos) de barreiras (paredes da arena), executando ações motoras específicas para cada caso. A solução foca na integração da percepção visual (detecção de cores) com a atuação motora (controle de juntas).

\section{Metodologia}
\paragraph{} A solução foi desenvolvida inteiramente com a API de \textit{scripting} Lua nativa do CoppeliaSim. A lógica principal reside em um \textit{child script} (script filho) anexado diretamente ao modelo do robô NAO, permitindo-lhe acesso direto aos seus próprios sensores e atuadores por meio de caminhos relativos.

\subsection{Ambiente de Simulação}
\paragraph{} O cenário é composto pelos seguintes elementos:
\begin{itemize}
    \item \textbf{Robô NAO:} O agente principal da simulação;
    \item \textbf{Arena (Jardim):} Um plano delimitado por quatro paredes. Estas paredes são de cor \textbf{verde} e servem como barreiras de navegação;
    \item \textbf{Alvos (Cubos):} Diversos objetos (cuboides) posicionados na arena, cada um com uma cor sólida distinta (ex: vermelho, azul, amarelo, preto, laranja);
    \item \textbf{Obstáculos (Plantas):} Objetos estáticos (`indoorPlant`) que servem como obstáculos visuais e de navegação.
\end{itemize}

\subsection{Arquitetura de Controle e Percepção}
\paragraph{} O sistema do robô é baseado em um ciclo de "percepção-ação" reativo.

\subsubsection{Sensores Utilizados}
\paragraph{} O principal, e único, sensor utilizado para a lógica de decisão foi o sensor de visão:
\begin{itemize}
    \item \textbf{\texttt{sphericalVisionRGB/Sensor}:} Um sensor de visão esférico (360 graus) anexado à cabeça do NAO. Este sensor é configurado no modo de detecção de "blobs" (manchas de cor). Ele é capaz de relatar o número de blobs detectados, bem como o tamanho e os valores RGB médios do maior blob em seu campo de visão.
\end{itemize}

\subsubsection{Atuadores (Motores) Controlados}
\paragraph{} Para executar as reações, o script controla os seguintes motores (juntas):
\begin{itemize}
    \item \textbf{\texttt{LShoulderPitch} e \texttt{RShoulderPitch}:} Motores responsáveis pelo movimento de subir e descer dos ombros. São usados para a ação de levantar os braços.
    \item \textbf{\texttt{HeadYaw}:} Motor responsável pelo movimento de "não" (rotação horizontal da cabeça). É usado para a ação de "desviar" da parede;
    \item \textbf{\texttt{LShoulderRoll} e \texttt{RShoulderRoll}:} (Opcional) Usados para auxiliar no posicionamento dos braços.
\end{itemize}

\subsection{Implementação da Lógica (Lua)}
\paragraph{} A lógica de controle é executada a cada passo da simulação dentro da função \texttt{sysCall\_actuation} do script principal do NAO.

\begin{enumerate}
    \item \textbf{Leitura da Percepção:} O script primeiramente consulta o sensor de visão para obter informações sobre o maior blob de cor detectado (usando \texttt{sim.handleVisionSensor} ou \texttt{sim.getVisionSensorBlobInfo});
    
    \item \textbf{Classificação da Cor:} Os valores RGB (normalizados de 0.0 a 1.0) retornados pelo sensor são passados para uma função auxiliar, \texttt{identificarCor(r, g, b)}. Esta função utiliza uma série de limiares \textit{(thresholds)} para classificar o RGB em uma \textit{string} de texto (ex: "vermelho", "verde", "preto", "indefinido").
    
    \item \textbf{Máquina de Estados (Lógica de Decisão):} O robô então entra em um dos três estados com base na cor e no tamanho do blob detectado:
    
    \begin{itemize}
        \item \textbf{Estado 1: Detecção de Barreira (Jardim):}
        \begin{itemize}
            \item \textbf{Condição:} Se \texttt{cor == "verde"} E \texttt{tamanho > 0.7} (ou seja, o blob verde ocupa mais de 70\% da visão, indicando proximidade com a parede);
            \item \textbf{Ação:} O robô executa a rotina \texttt{abaixarBracos()} e ativa um movimento oscilatório no motor \texttt{\textit{HeadYaw}} (simulando um "não") para sinalizar a detecção da barreira.
        \end{itemize}
        
        \item \textbf{Estado 2: Detecção de Cubo (Alvo):}
        \begin{itemize}
            \item \textbf{Condição:} Se \texttt{cor ~= "indefinido"} E \texttt{cor ~= "verde"};
            \item \textbf{Ação:} O robô para o movimento da cabeça (\texttt{HeadYaw = 0}) e executa a rotina \texttt{levantarBracos()}, movendo as juntas \texttt{LShoulderPitch} e \texttt{RShoulderPitch} para uma posição elevada.
        \end{itemize}
        
        \item \textbf{Estado 3: Ocioso (Procurando):}
        \begin{itemize}
            \item \textbf{Condição:} Se nenhuma cor for detectada (\texttt{cor == "indefinido"}) ou se for um blob verde pequeno;
            \item \textbf{Ação:} O robô retorna à sua posição neutra, executando \texttt{abaixarBracos()} e centralizando a cabeça.
        \end{itemize}
    \end{itemize}
\end{enumerate}

\section{Resultados e Discussão}
\paragraph{} O controlador implementado demonstrou ser eficaz em sua tarefa principal. O robô foi capaz de distinguir com sucesso os cubos-alvo coloridos e a barreira verde do jardim.

A utilização de uma heurística baseada no tamanho do blob (\texttt{tamanho > 0.7}) provou ser uma solução simples e funcional para a detecção de paredes, dispensando a necessidade de sensores de proximidade (como sonares ou Infravermelho) para esta tarefa específica.

Um desafio notável na implementação é a calibração da função \texttt{identificarCor}. Os limiares de RGB são altamente sensíveis às condições de iluminação da cena no CoppeliaSim. Mudanças na iluminação ambiente exigiriam uma recalibração dos valores para garantir a correta identificação das cores.

\section{Conclusão}
\paragraph{} Este trabalho demonstrou com sucesso a implementação de um sistema de percepção e reação para o robô NAO no CoppeliaSim. Utilizando apenas um sensor de visão e scripts Lua, o robô foi capaz de identificar objetos por cor e reagir de forma diferenciada a alvos e barreiras.

Como trabalhos futuros, a solução poderia ser expandida para incluir:
\begin{itemize}
    \item A navegação real e o desvio de obstáculos (usando sensores de proximidade para as "plantas");
    \item A adição de síntese de voz (TTS) para que o robô fale a cor detectada, completando o requisito original do projeto;
    \item A implementação de uma máquina de estados mais robusta, para uma busca ativa pelos cubos.
\end{itemize}

% --- REFERÊNCIAS ---
\begin{thebibliography}{9}

    \bibitem{coppelia}
    Coppelia Robotics. (2025).
    \textit{CoppeliaSim User Manual}.
    Disponível em: \url{https://www.coppeliarobotics.com/helpFiles}

    \bibitem{nao_doc}
    SoftBank Robotics. (2025).
    \textit{NAO V6 Documentation}.
    Disponível em: https://www.gov.br/anatel/pt-br

\end{thebibliography}

\end{document}
% --- FIM DO DOCUMENTO ---
```
