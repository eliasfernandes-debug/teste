# teste

## Como ajustar imagens na Wiki do Azure DevOps

### Usando a tag HTML `<img>`

Para inserir e redimensionar uma imagem usando HTML, utilize a tag `<img>` com os atributos `src`, `alt`, `width` e/ou `height`:

```html
<img src="URL_DA_IMAGEM" alt="Descrição da imagem" width="300" />
```

Exemplo fixando largura e altura:

```html
<img src="https://exemplo.com/imagem.png" alt="Minha imagem" width="400" height="200" />
```

### Usando a sintaxe Markdown do Azure DevOps

O Azure DevOps Wiki suporta uma extensão de Markdown para redimensionar imagens diretamente:

```markdown
![Descrição da imagem](URL_DA_IMAGEM =LARGURAxALTURA)
```

Exemplos:

```markdown
![Logo](https://exemplo.com/logo.png =200x100)

![Banner](https://exemplo.com/banner.png =500x)
```

> **Nota:** Na sintaxe `=LARGURAxALTURA`, você pode omitir a altura (`=500x`) para manter a proporção original da imagem.

### Dicas

- Prefira definir apenas a **largura** (`width`) e deixar a altura proporcional para evitar distorções.
- Use caminhos relativos para imagens que estejam no próprio repositório Wiki.
- Imagens hospedadas externamente requerem que a URL seja pública e acessível.
