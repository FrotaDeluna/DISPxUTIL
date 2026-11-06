# Gestão de Frota — Painel Gerencial

Dashboard em HTML, CSS e JavaScript para acompanhar disponibilidade física e capacidade real de utilização a partir das abas diárias da planilha **DISPONIBILIDADE - UTILIZAÇÃO.xlsx**.

## O que está implementado

- Leitura de abas com nome no padrão `DD.MM`, ignorando `LOGISTICS` e outras abas sem data.
- Cabeçalho na linha 2; dados a partir da linha 3; placa, base e status identificados pelas colunas A, B e C.
- Identificador único por placa em cada dia; linhas vazias ignoradas e duplicidades não contabilizadas.
- Disponibilidade física calculada como frota total menos registros cujo status contém `MANUTENÇÃO`.
- Capacidade operacional calculada com status explicitamente disponíveis; status desconhecidos são tratados como indisponíveis até classificação.
- Configuração local de status, filtros por período/dia/base/locadora, gráficos, histórico, exportação Excel e detalhamento das placas após importação local.
- Persistência do histórico e do detalhamento local no IndexedDB do navegador.
- Resumo inicial real, derivado das 26 abas diárias do arquivo recebido, sem dados fictícios e sem relação de placas no arquivo publicado.

## Arquivos

- `index.html`: interface principal.
- `index.single.html`: versão consolidada, com CSS, JavaScript e resumo inicial incorporados em um arquivo.
- `style.css`: identidade visual responsiva.
- `script.js`: eventos, filtros, renderização, IndexedDB, importação e verificação de versão publicada.
- `js/core.js`: identificação de datas, normalização de status e cálculos.
- `data/initial-summary.js` e `data/initial-summary.json`: posição histórica agregada usada para abrir o painel já preenchido e atualizar a versão online.
- `scripts/build_public_data.cjs`: processamento da planilha para gerar dados agregados para o site.
- `.github/workflows/deploy.yml`: automação do GitHub Actions.

## Utilização local

1. Extraia o ZIP mantendo a estrutura de pastas.
2. Para melhor compatibilidade com o armazenamento persistente, abra um terminal nesta pasta e execute `python -m http.server 8000`; depois acesse `http://localhost:8000`. Também é possível abrir `index.html` diretamente no Chrome ou Edge.
3. O painel já carrega os indicadores agregados da planilha usada na construção desta versão. Para incorporar uma planilha atualizada, clique em **Importar planilha** e selecione o novo Excel.
4. O histórico e os dados detalhados ficam persistidos no IndexedDB daquele navegador. A exportação gera um Excel com os indicadores do período filtrado.

A conexão com a internet é necessária para carregar SheetJS e Chart.js pelos CDNs usados nesta versão. Para um ambiente sem internet, baixe essas bibliotecas e hospede-as localmente, alterando as referências no `index.html`.

**Limite do navegador:** o painel aberto localmente não consegue detectar sozinho novas abas de um arquivo Excel que está no computador. É necessário importar novamente a planilha.

## Publicação e atualização automática pelo GitHub

### Estrutura esperada no repositório

Mantenha o código na raiz e coloque a planilha de origem em:

```text
source/DISPONIBILIDADE - UTILIZAÇÃO.xlsx
```

O arquivo de origem não fica dentro da pasta publicada (`dist`). Por padrão, o script gera no site somente totais agregados por dia, status, base e locadora; a relação de placas não é incluída. O detalhamento fica disponível no navegador depois de uma importação local.

Caso a empresa tenha GitHub Enterprise Cloud e o site esteja efetivamente publicado com visibilidade **Private**, é possível habilitar o detalhamento online criando também a variável `PUBLISH_PLATE_DETAILS=true`. O script bloqueia essa opção se `ENABLE_PRIVATE_PAGES=true` não estiver definido. Nunca habilite esse detalhamento em um Pages público.

### Passos para ativar

1. Crie um repositório **privado** no GitHub e envie os arquivos deste projeto.
2. Coloque a planilha de origem em `source/DISPONIBILIDADE - UTILIZAÇÃO.xlsx` e faça o commit.
3. Em **Settings → Pages**, verifique a opção de visibilidade do site. A página só deve ser habilitada quando estiver configurada como **Private** e disponível no plano/organização GitHub correspondente.
4. Em **Settings → Secrets and variables → Actions → Variables**, crie a variável `ENABLE_PRIVATE_PAGES` com valor `true` somente depois de confirmar que a publicação do Pages será privada.
5. Opcionalmente, e somente com Pages privado confirmado, crie `PUBLISH_PLATE_DETAILS=true` para disponibilizar também as placas no site. Sem essa variável, o site recebe somente os totais agregados.
6. Em **Settings → Pages → Build and deployment**, selecione **GitHub Actions**.
7. Faça um commit da planilha sempre que houver novas abas ou correções. O workflow processará todas as abas diárias, recalculará os indicadores e publicará a versão atualizada.

O navegador consulta `data/version.json` a cada cinco minutos. Quando detectar nova versão publicada, recarrega o painel. O botão **Atualizar dados** também força a verificação.

### Aviso essencial de privacidade

O GitHub Pages normalmente publica o site na internet, mesmo que o repositório seja privado. A publicação privada do Pages depende de GitHub Enterprise Cloud e da configuração de visibilidade disponível para a organização. **Não habilite `ENABLE_PRIVATE_PAGES=true` sem verificar a visibilidade privada do site.**

Os dados agregados também podem ser informação operacional confidencial. Caso sua conta não ofereça Pages privado, mantenha a publicação automática desabilitada e use o painel local, ou hospede o site atrás de autenticação (por exemplo, Cloudflare Access ou outra solução corporativa). Não coloque tokens pessoais do GitHub no JavaScript do navegador.

## Regras dos indicadores

- **Disponibilidade física:** `frota total - manutenção`. O contador de manutenção considera um veículo uma única vez mesmo quando o status tem complemento, por exemplo `MANUTENÇÃO / SEM DRIVER`.
- **Capacidade de utilização:** `frota total - indisponíveis para operação`. Apenas `ATIVA`, `RESERVA / ATIVA` e `ATIVA - RESERVA TITULAR` iniciam como disponíveis. Motivos conhecidos são indisponíveis; novos status ficam pendentes e são tratados como indisponíveis até classificação.
- **Indisponíveis por outros motivos:** indisponíveis totais da operação menos veículos em manutenção. Cada placa é contada uma vez por dia.
- Os filtros de base e locadora recalculam os indicadores a partir dos registros daquela dimensão.
- A data das abas é inferida pela sequência cronológica das abas e pelo ano mais próximo da data atual; em uma virada de ano, a sequência `DD.MM` permite reconhecer o retorno de dezembro para janeiro.

## Testes

Execute `npm test` para validar a série real incluída nesta entrega e cenários controlados de duplicidade, status combinado, inclusão de uma nova aba, atualização de uma data existente, classificação editável e virada de ano.

## Validação da planilha recebida

O arquivo usado para gerar os dados iniciais contém 26 abas diárias, entre 01/10/2026 e 06/11/2026. Na aba `06.11`, foram identificadas 374 placas únicas. Pela regra de manutenção (incluindo a variação `MANUTENÇÃO (RESERVA BIPANDO)`), a posição consolidada é:

| Indicador | Resultado em 06/11/2026 |
|---|---:|
| Frota total | 374 |
| Veículos em manutenção | 37 |
| Disponibilidade física | 337 (90,11%) |
| Indisponíveis para operação | 84 |
| Capacidade de utilização | 290 (77,54%) |
| Indisponíveis por outros motivos | 47 |

Esses valores são calculados a partir das placas únicas da aba diária e não são números fixos no código.

## Arquivo consolidado

`index.single.html` é a versão de arquivo único com CSS, JavaScript de aplicação, motor de cálculo e resumo inicial incorporados. As bibliotecas SheetJS e Chart.js ainda são carregadas por CDN. Para automação no GitHub, use a estrutura completa deste projeto, pois o workflow e o script de processamento precisam permanecer separados.
