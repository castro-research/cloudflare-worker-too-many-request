# Introdução

Um detalhado relato de como resolvi um problema na API que utilizava o serviço browser rendering Worker (Cloudflare)

TLDR: Temos uma API Ruby on Rails que recebe uma request com um body, avaliamos, e repassamos ao Worker da Cloudflare para gerar um PDF, e essa API parou de funcionar "do nada".

# Início

No meio das férias, uma notificação:

<img src="https://github.com/user-attachments/assets/9e7219b4-3e9f-4627-a68b-1ecbf936590b" />

Minha primeira reação: Vamos ao Sentry ver o que tá acontecendo.

Vejo 170 tentativas de gerar uma página com puppeteer.

<img src="https://github.com/user-attachments/assets/dd7d5cb1-3439-4e50-b6cc-60e62ef2bbe3" />


O primeiro problema é que, como eu estava usando Typescript, eu tinha a versão transpilada, mas sem [sourcemap](https://developers.cloudflare.com/workers/observability/source-maps/) (que ainda está em BETA ), e a flag do upload_source_maps não estava ativa. (Acho que era para ser padrão para novos projetos Typescript ou já identificar, sei lá.)

Então, sem o Macbook em mãos, fui olhar os logs pelo celular (que não diz nada relevante), já que não tenho o stacktrace e onde ocorreu o problema.

### o que tinha acontecido (Na minha cabeça)

Recebi muitas requisições, tentou escalar para mais workers do que devia, deu Out of memory, e parou de receber requisição por um tempo, dando erro 503 também.

### O que eu assumi:

Já que o limite do plano pago são 500 workers, Para cada requisição, um Worker abre um headless browser, faz o que tem que fazer, fecha o browser, termina e encerra.

# Descobrindo o problema

Já com o Macbook em mãos, e sabendo que eu não tinha stack trace, e nem log, passei o código para JS Module, ao invés de Typescript, para não ter que testar source maps ainda em Beta.

Como o projeto tem testes de integração, rodei eles e tudo passou.

Subi uma versão de preview remoto, Escrevi um script separado para fazer 10 requisição ao mesmo tempo (via callbacks), e voilà, erro 429 a vista. Sem transpile, o debugger ficou muito mais fácil.

Fui a documentação procurar por [Rate limit](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/), e dizia a mesma coisa na parte dos planos, e eu poderia até editar rate limit baseado no user, localização, etc...

Então pensei que tinha alguma parte do Cloudflare security, onde eu não tinha acesso.

Acessa aqui, acessa ali, e encontro na documentação que o [limite do Browser Rendering](https://developers.cloudflare.com/browser-rendering/platform/limits/), é diferente do Worker.

>> Two new browsers per minute per account.
>> Two concurrent browsers per account

Bingo! Eu não podia mais que 2 sessões do browser ao mesmo tempo.

# Cavando mais fundo

Não era só isso, as informações estavam divididas em pequenas partes de boas práticas. Então o trabalho agora, antes de ir pro código mesmo, era juntar esses pedaços:

1 - A nova instância do browser tinha que estar dentro de um try/catch para não ter sessões perdidas / crashes

Isso eu já tinha feito

2 - Omitir o `browser.close`, era necessário para manter o `keep-alive`, e atender mais requisições em diferentes workers sem precisar de abrir uma nova instância.

Como eu tinha um outro entendimento, eu sempre fechava, então nunca atendia mais que 2 requisições  ao mesmo tempo.

3 - Você tem acesso aos workers das sessões dos browsers.

Pronto, não sabia disso também, e podia verificar se já tinha algum aberto e fazer um `connect` ao invés de `launch`, e fazer meio que um `round-robin`.

4 - Você deve fechar o browser QUANDO ocorrer um erro, mas deve fazer APENAS `disconnect` da sessão, se conseguir fazer o que tem para fazer, e liberar para outra request.

5 - Você pode ter uma função recursiva para fazer o retry da conexão ao browser.

Até o 4, você resolve parte do problema de manter 2 sessões em aberto e ficar com `Too Many Request` preso por um tempo, e depois bloquear de novo após outra requisição. Mas ainda terá alguns erros 429, por que em alguma hora, vais ter mais concorrências, e os Workers estarão ocupados.

Esta implementação em específico não está na documentação e poderia ser implementada pelo Rails. Porém, implementei o retry de uma forma simples, onde ele tentaria procurar outro browser disponível, e somente retorno erro, quando realmente o worker não conseguir fazer o que tem que ser feito ou der timeout.

Por que não tivemos esse problema antes?

Por que ainda estamos testando uma funcionalidade, e nunca foi necessário fazer mais que algumas request em um espaço de tempo.

# Conclusão

Implementei um teste que faz algumas request ao mesmo tempo e todos tiveram status 200

<img src="https://github.com/user-attachments/assets/905fe43b-0e41-48c4-a20f-5d8fc17e40f0" />

# Outros desafios

Já que temos 2 workers limitados de browser rendering, o conteudo do html importa demais, já que se for muito grande, vai ser maior tempo esperando processar, e então o timeout será inevitável.