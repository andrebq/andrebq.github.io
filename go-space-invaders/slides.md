<!-- $theme: default -->

Space Invaders usando Go and SDL2
===

André Luiz Alves Moraes

```bash
➜  go-space-invaders git:(master) ✗ whoami
programmer since 2001 (+/-)
go programmer since Nov/2009
```

Code available at [github.com/andrebq/space-invaders](https://github.com/andrebq/space-invaders)

---

Introdução
===

---

O que é SDL
===

SDL (Simple Directmedia Layer) é uma biblioteca para desenvolvimento de aplicações gráficas e interativas.

Ela expõe uma API para: Windows, Linux, Mac OS X, Linux, iOS, Android

[https://www.libsdl.org/]

API low-level usada por diversos jogos da Valve e HumbleBundle

---

APIs disponibilizadas
===

* Video
* Input Events
* Force Feedback
* Audio
* File I/O Abstraction
* Shared Object Support
* Threads
* Timers
* Endian Independence
* Power Management

---

Motivos para usar SDL2
===

* Flexível o suficiente para ser utilizada na criação de jogos e/ou aplicações
* Possui três modos de rasterização: CPU render, CPU render usando memória de vídeo, OpenGL
* Permite utilizar apenas os módulos necessários (audio, video, input, etc...)
* Comunidade ativa e muita documentação (inclusive a documentação oficial que é excelente)

---

Engine criada
===

Engine baseada no modelo CES (Component-Entity-System)

* Component: contém os dados relacionados a determinada funcionalidades (em alguns casos comportamento também)
* Entity: é um agregado de components
* System: manipula entidades que possuam um conjunto específico de componentes

Exemplo: O sistema de renderização manipula entitades que possuem um componente de renderização, tais entidades
podem possuir componentes de simulação de física porém o sistema ignora tais componentes.

---

Organização do código
===

* main package: contém chamadas para inicializar o SDL e a engine
* ces package: contém a implementação base da engine (define as interfaces e o loop inicial)
* ces/input package: contém a parte da engine que trata input do usuário e despacha os eventos para as entidades
* ces/render package: contém o sistema que aloca os recursos gráficos e despacha as renderizações
* ces/sfx package: contém os componentes/entidades do sistema de som
* game package: implementa os diversos componentes e contém a lógica do jogo em si (player, stages, inimigos)

---

Pontos importantes
===

* A maioria dos sistemas considera a coordenada (0,0) como o canto superior esquerdo, porém para jogos normalmente considera-se (0,0) como
o canto inferior esquerdo
* No OpenGL no geral (0,0) é o centro da tela, mas isso é facilmente alterado utilizando matrizes
* 90% das vezes em jogos deve-se utilizar SDLTexture no lugar de SDLSurface
* É importante reduzir a quantideade de texturas utilizadas. Uma textura gigante com muitas imagens é melhor do que muitas texturas pequenas
* Todos os cálculos de física devem levar em conta o frame-rate
* Multiplayer não é fácil!

---

Demo time
===

Vamos rezar e tentar jogar!

---

Live-coding Incluindo música de fundo
===

Vamos rezar mais ainda!

---

Copyright
===

bomb_explosion.wav
- Author: Luke.RUSTLTD
- Download link: https://opengameart.org/content/bombexplosion8bit
- Description: A bomb sound done in CFXR

explosion-1.wav
- Author: jobromedia
- Download link: https://opengameart.org/content/jbm-sound-effects-pack-1
- Description: Are you looking for something reasonably oldskool? Something that was rad as heck back in the early 80's? Congrats, you've just found it! JBM Sound effects pack 1 contains 70 different sound effects arranged in 9 categories.

deep-hunt_rafael-oliveira
- Author: Rafael Oliveira [http://rafaelmo.com.br/]
- Download link: https://soundcloud.com/orafael/title-deep-hunt-rafael-oliveira
- Description: Soundtrack idea for a game.

---

# Obrigado!

---

### Perguntas?
