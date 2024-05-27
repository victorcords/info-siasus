[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/victorcords/info-siasus/blob/main/info_siasus.ipynb)

# Info-SIASUS
Análise do banco de dados do Sistema de Informação Ambulatoriais do DATASUS.

## Introdução
A extração e tratamento dos dados presentes nos bancos de dados do DATASUS podem fornecer informações precisas e orientar condutas e políticas públicas. O Sistema de Informações Ambulatoriais (SIA) apresenta uma estrutura e conjunto de dados que, agreagados da forma correta, podem ser úteis tanto cientificamente quanto estrategicamente, como Ferré et al. demonstrou e exemplificou métodos para extrair coortes dos bancos de dados do SIA do DATASUS (e de outros sistemas), o que torna possível a análise de diversos elementos, dentre eles, a análise da dispensação de medicamentos, quantitativo de usuários desses medicamentos e o valor gasto. Dessa forma, este projeto tem como objetivo realizar uma análise dos medicamentos (dispensação, quantitativo de usuários e valor gasto) utilizados conforme o CID previamente especificado em determinada Unidade da Federação em um período de tempo. A vantagem deste projeto é a utilização do pacote `microdatasus` para facilitar o processo de extração e transformação. 

Este projeto foi desenvolvido em `R` no ambiente do **Google Colab**. As bibliotecas utilizadas foram:
1. [Microdatasus](https://github.com/rfsaldanha/microdatasus);
2. [Gsubfn](https://cran.r-project.org/web/packages/gsubfn/index.html);
3. [RCurl](https://cran.r-project.org/web/packages/RCurl/index.html);
4. [Stringr](https://cran.r-project.org/web/packages/stringr/index.html);
5. [Readr](https://cran.r-project.org/web/packages/readr/index.html);
6. [Forecast](https://cran.r-project.org/web/packages/forecast/index.html);
7. [Tseries](https://cran.r-project.org/web/packages/tseries/index.html).

Os arquivos `.csv` adicionais utilizados estão localizados no diretório **bps_2018_2023** deste repositório. Nesta análise, foi utilizado como exemplo o estado de Goiás. O período de referência foi o ano de 2023, apenas nos meses de novembro e dezembro. Os resultados foram filtrados para Doença de Crohn (CID-10 K50.0, K50.1 e K50.8), com os imunobiológicos descritos no [Protocolo Clínico e Diretrizes Terapêuticas para Deonça de Crohn](https://www.gov.br/conitec/pt-br/midias/protocolos/portaria_conjunta_14_pcdt_doenca_de_crohn_28_11_2017-1.pdf). Na sessão "Processamento PA" no Notebook do Google Colab, há a descrição dos medicamentos e seus respectivos códigos no Sistema de Gerenciamento da Tabela de Procedimentos, Medicamentos e OPM do SUS (SIGTAP). Os algoritmos de Extração, Transformação e Carga dos bancos de dados do DATASUS foram validados e explorados em outros trabalhos, como demonstrado por [Ferré](https://rpubs.com/labxss/858478). 

## Dataset
Os dados gerados a partir da extração e processamento dos bancos de dados incluíram as seguintes variáveis nas colunas:

| CAMPO  | DESCRIÇÃO |
| ------------- | ------------- |
| PA_AUTORIZ  | Número da APAC ou número de autorização do BPA-I, conforme o caso  |
| PA_CMP	  | Data da Realização do Procedimento / Competência (AAAAMM)  |
| PA_MVM	  | Data de Processamento / Movimento (AAAAMM)  |
| PA_CIDPRI	  | CID Principal (APAC ou BPA-I)  |
| PA_CIDSEC	  | CID Secundário (APAC)  |
| PA_PROC_ID		  | Código do Procedimento Ambulatorial  |
| PA_SEXO  | Sexo do paciente  |
| PA_IDADE  | Idade do paciente em anos  |
| PA_MUNPCN	  | Código da Unidade da Federação + Código do Município de residência do paciente ou do estabelecimento  |
| PA_QTDAPR  | Quantidade Aprovada do procedimento  |
| uf_processamento  | Código da Unidade da Federação de processamento  |
| AP_CNSPCN  | Número do CNS (Cartão Nacional de Saúde) do paciente (codificado)  |

Tabela de autoria própria. CID: Classificação Internacional de Doenças, 10ª Revisão. uf – sigla da Unidade da Federação; AAAA – ano da competência; MM – mês da competência. Fonte: Informe Técnico do Sistema de Informações Ambulatoriais (SIA/SUS). Dados disponibilizados pelo Departamento de Informática do Sistema Nacional de Saúde (DATASUS - www.datasus.gov.br). Ministério da Saúde, Brasil.

## Utilização 
Carregue o arquivo `info-siasus.ipynb` no ambiente do Google Colab. Diversos campos são modificáveis conforme o escopo da análise. Alguns campos serão discutidos a seguir, como forma de facilitar o entendimento dos parâmetros e argumentos. 
### Extraindo os dados
No exemplo do Notebook, foram extraídos os dados do ano de 2023, dos meses de novembro e dezembro, para UF Goiás, no SIA-PA. 
```r
#=================[DADOS SIA-PA]=============

siapa_go <- fetch_datasus(
                       year_start = 2023, year_end = 2023, 
                       month_start = 11, month_end = 12,
                       uf = "GO",
                       information_system = "SIA-PA")

#============================================
```
O vetor `siapa_go` pode ser nomeado de acordo com a preferência do usuário. Lembre-se apenas de substituir as posteriores referências ao vetor. Utilize o manual para utilização do pacote Microdatasus na [Wiki do projeto](https://github.com/rfsaldanha/microdatasus/wiki). 

A seguir, foram extraídos os dados do SIA-AM.
```r
#=================[DADOS SIA-AM]=============

siaam_go <- fetch_datasus(
                       year_start = 2023, year_end = 2023,
                       month_start = 11, month_end = 12,
                       uf = "GO",
                       vars = c("AP_AUTORIZ", "AP_PRIPAL", "AP_CIDPRI", "AP_CNSPCN"),
                       information_system = "SIA-AM")

#============================================
```
O vetor `siaam_go` também pode ser nomeado de acordo com a preferência do usuário. As colunas foram pré-selecionadas utilizando o argumento `vars`. 

### Processamento dos dados
Para filtrar os dados de acordo com o [CID da doença](https://cid10.com.br/) e o procedimento descrito no SIGTAP, altere `cids` e `sigtap`. Estes parâmetros são utilizados para o processamento dos dados no `data.frame`. 

```r
#=================[FILTROS]=================

cids = c(
  "K500",
  "K501",
  "K508"
) # CID-10

sigtap = c(
  "0604380011", # ADALIMUMABE 40 MG INJETÁVEL (FRASCO AMPOLA)
  "0604380127", # ADALIMUMABE 40 MG INJETAVEL (POR SERINGA PREENCHIDA)
  "0604380135", # ADALIMUMABE 40 MG INJETÁVEL (POR SERINGA PREENCHIDA) (BIOSSIMILAR B)
  "0604380046", # INFLIXIMABE 10 MG/ML INJETAVEL (POR FRASCO-AMPOLA COM 10 ML)
  "0604380054", # INFLIXIMABE 10 MG/ML INJETAVEL (POR FRASCO-AMPOLA COM 10 ML)
  "0604380119", # INFLIXIMABE 10 MG /ML INJETÁVEL (POR FRASCO-AMPOLA COM 10 ML) (BIOSSIMILAR A)
  "0604380070"  # CERTOLIZUMABE PEGOL 200 MG/ML INJETÁVEL (POR SERINGA PREENCHIDA)
) # Biológicos

```
Após isso, foram criadas tabelas vazias no formato `data.frame` contendo as variáveis desejadas. As variáveis desejadas foram analisadas e selecionadas a partir do dicionário de dados do Sistema de Informações Ambulatoriais do SUS. Os dados foram processados conforme a estrutura do `data.frame` estabelecido. Posteriormente, os dados do SIA-PA e SIA-AM, por meio do número de autorização, em geral, **PA_AUTORIZ** ou **AP_AUTORIZ**, foram complementados ao arquivo principal. A tarefa é fundamental para computar o número de usuários com quantidade aprovada de dado procedimento, sobretudo a partir do arquivo de medicamentos de prefixo AM. Ou seja, a junção ocorre segundo o código da autorização.

## Contribuições
As contribuições são o que tornam a comunidade de código aberto um lugar incrível para aprender, inspirar e criar. Quaisquer contribuições que você faça serão muito apreciadas. 

## Referências

CAROLINA ANDRADE OLIVEIRA DIBAI; JULIANA ALVARES-TEODORO; FELIPE FERRÉ; MARIANO RUAS, C. Avaliação da eficiência na aquisição e distribuição de medicamentos do Componente Básico da Assistência Farmacêutica. JORNAL DE ASSISTÊNCIA FARMACÊUTICA E FARMACOECONOMIA, [S. l.], v. 8, n. 4, 2023. DOI: 10.22563/2525-7323.2023.v8.n.4.p.23-36. Disponível em: https://ojs.jaff.org.br/ojs/index.php/jaff/article/view/642. Acesso em: 23 mai. 2024.

FERRÉ, F. Produto 3 - Relatório técnico contendo análise da projeção do quantitativo de usuários, unidades dispensadas e valor gasto apresentados na sala de situação aberta de medicamentos biológicos cujo acesso é regulado por PCDT. Brasil: RStudio/RPubs, 2022. Disponível em: <https://rpubs.com/labxss/858478>. Acesso em: 23 mai. 2024.

FERRÉ, F.; DE OLIVEIRA, G.; DE QUEIROZ, M.; GONÇALVES, F.. Sala de Situação aberta com dados administrativos para gestão de Protocolos Clínicos e Diretrizes Terapêuticas de tecnologias providas pelo SUS. In: SIMPÓSIO BRASILEIRO DE COMPUTAÇÃO APLICADA À SAÚDE (SBCAS), 20. , 2020, Evento Online. Anais [...]. Porto Alegre: Sociedade Brasileira de Computação, 2020 . p. 392-403. ISSN 2763-8952. DOI: https://doi.org/10.5753/sbcas.2020.11530.

FERRÉ, F. Curso de linguagem R com dados do SUS. Disponível em: <https://github.com/labxss/curso_r>. Acesso em: 23 mai. 2024.

FERRÉ, F. Sala Aberta de Inteligência em Saúde - Protocolos Clínicos e Diretrizes Terapêuticas. Disponível em: <https://github.com/labxss/sabeis_pcdt>. Acesso em: 21 mai. 2024.


SALDANHA, R. DE F.; BASTOS, R. R.; BARCELLOS, C.. Microdatasus: pacote para download e pré-processamento de microdados do Departamento de Informática do SUS (DATASUS). Cadernos de Saúde Pública, v. 35, n. 9, p. e00032419, 2019.

## Contato
📩 Victor Cordeiro Simão - victorcordeiro@discente.ufg.br

💡 Link do Projeto - [https://github.com/victorcords/info-siasus](https://github.com/victorcords/info-siasus)

