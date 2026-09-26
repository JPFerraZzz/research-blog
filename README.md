# research-blog

Fonte do meu blog de research pessoal, publicado em [jaoresearch.duckdns.org](https://jaoresearch.duckdns.org).

Escrevo write-ups técnicos sobre o que investigo (pentest, análise de APIs,
segurança de LLMs), como forma de documentação verificável do meu trabalho.
Cada commit é assinado com GPG, o selo "Verified" neste repositório confirma
que o conteúdo é meu e não foi alterado depois de publicado.

## Stack

- [Hugo](https://gohugo.io) (gerador de site estático)
- Tema [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- Hospedado num servidor próprio via Nginx + Let's Encrypt

## Estrutura

content/posts/ write-ups, um ficheiro Markdown por post
themes/PaperMod/ tema, incluído como submódulo
hugo.toml configuração do site


## Correr localmente

```bash
git clone --recurse-submodules https://github.com/JPFerraZzz/research-blog.git
cd research-blog
hugo server -D
```

## Licença

Sem licença explícita, todo o conteúdo é meu por defeito de copyright. Se
quiseres citar ou reutilizar algo, contacta-me.
