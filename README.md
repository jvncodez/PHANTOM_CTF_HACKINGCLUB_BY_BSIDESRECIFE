# Phantom — Writeup e Exploit (HackingClub / BSides Recife)

Resolução completa da máquina Phantom (portal NFS-e em PHP), do CTF da BSides Recife (HackingClub). A gente travou nela durante o evento e finalizou depois do CTF, com calma.

A flag exige encadear quatro vulnerabilidades, cada uma destravando a seguinte:

> Auth bypass (método HTTP) › SSRF/LFI › PHAR deserialization (POP chain) › RCE

## Conteúdo

* [`WRITEUP-Phantom.md`](./WRITEUP-Phantom.md): passo a passo com o raciocínio real e os becos sem saída.
* [`phantom_exploit.py`](./phantom_exploit.py): exploit automatizado (só requer `python3`, stdlib).

## Uso

```bash
python3 phantom_exploit.py            # alvo padrão: phantom.hackingclub.com.br
python3 phantom_exploit.py https://alvo
```

```
[+] auth bypass via PATCH > sessao admin
[+] LFI/SSRF confirmados (li /etc/passwd via file://)
[+] phar enviado: /var/www/shared/logos/<md5>.png
[+] RCE confirmada em http://renderer/sh_<id>.php > uid=33(www-data)
[+] FLAG: hackingclub{...}
```

O exploit redescobre tudo em tempo de execução: testa qual método loga, confirma o LFI, lê o código fonte via LFI, monta o phar em Python puro e procura a flag no filesystem. Nada de valores chumbados.

## A cadeia, em resumo

| # | Vulnerabilidade | Onde | Resultado |
|---|---|---|---|
| 1 | Auth bypass por método HTTP | `login.php` (`PUT`/`PATCH`) | sessão de admin sem senha |
| 2 | SSRF e LFI (blocklist fraca) | `api/cnpj.php` (`api=`) | leitura de arquivo e acesso interno |
| 3 | Fonte exposto por volume compartilhado | `/var/www` (portal e renderer) | senha real e gadgets da POP chain |
| 4 | PHAR deserialization e escrita arbitrária | `render.php` (`logo_path=phar://`) | webshell e RCE |

## Aviso de uso

Material educacional, publicado após o término do CTF. Use somente em ambientes de CTF ou laboratório autorizados. Não aponte o exploit para sistemas sem permissão explícita.

## Créditos

Apoio de ferramental: Agente VION, agente de pentest desenvolvido internamente para otimizar recon e automação.

João Silva e equipe, pesquisadores de segurança e hackers éticos.
