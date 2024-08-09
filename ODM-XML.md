---
title: "XML na Prática: Como lidar com documento ODM-XML usando Python"
date: "2024-08-07"
slug: "xml-na-pratica"
image: "https://i.ibb.co/LkdzH1b/Untitled.png"
---

## Contexto

Em estudos clínicos, é comum trabalharmos com conjuntos de dados que podem vir em variados formatos, dependendo do que o sistema [EDC](https://en.wikipedia.org/wiki/Electronic_data_capture) escolhido consegue oferecer. Entretanto, em algumas situações, será preciso extrair um conjunto de dados completo, de acordo com o que as entidades regulatórias exigem nas submissões de aprovação do uso de medicamentos e vacinas, principalmente. A partir disso, não há muita escolha; sempre aparecerá como a opção mais facilitada um documento XML. Para quem está acostumado a lidar com todo tipo de dado, talvez um pouco de esforço terá de ser feito pra transformar isso em algo que possa ser usados em várias análises. E para quem está acostmado a lidar apenas com dados estruturados (planilhas), o desespero bate. E agora? O que fazer?

Neste artigo, veremos um pouco dos segredos que escondem um documento ODM-XML, além de como podemos extrair esses segredos, e produzir uma planilha com algo que faça sentido na hora das análises de dados de estudos clínicos.

## Por que XML?

Num apanhado geral, [há alguns (bons) motivos para se usar XML ao lidar com dados](https://aws.amazon.com/what-is/xml/). Exemplificando:

- **É possível criar um arquivo que pode ser utilizado com as mais variadas ferramentas de manipulação de dados**. Muitas tecnologias, inclusive as mais recentes, já vêm com um suporte **nativo** ao XML, o que facilita muito o trabalho de quem irá lidar com os dados.
- **Mantém a integridade dos dados**, o que é um aspecto vital na hora de lidar com base de dados. Isso é feito a partir da concentração de aspectos como metadados e dados dentro de um só arquivo, fazendo com que todos os recursos estejam à disposição sem precisar importar outros arquivos (e isto iremos explorar de forma um pouco mais detalhada ao longo do artigo)<sup id="1">[1](#um)</sup>.
- **A organização e categorização dos dados é mais eficiente**, fazendo com que a busca por uma determinada informação dentro de um arquivo XML seja mais rápida e menos, digamos, "dolorosa".

## O que é ODM?

[ODM](https://www.altexsoft.com/blog/cdisc-standards/) é a sigla para *Operational Data Model*. Segundo o [CDISC](https://www.cdisc.org/standards/data-exchange/odm), "é um formato que pode ser utilizado para armazenar, intercambiar entre sistemas de gestão de dados, bem como para armazenar os dados, metadados, e dados administrativos referentes a um estudo clínico". Para algumas entidades regulatórias de saúde, como a FDA dos Estados Unidos <sup id="2">[2](#dois)</sup> <sup id="3">[3](#três)</sup> <sup id="4">[4](#quatro)</sup> <sup id="5">[5](#cinco)</sup>, o CDISC sugeriu este formato como o padrão para arquivamento de dados, por sua capacidade de carregar todas as informações pertinentes a estudos clínicos em um só arquivo intercambiável, quando preciso.

O modelo ODM oferece um modelo XML que facilita a captura de dados clínicos; a partir de um esquema como o da figura abaixo, podemos dividir o documento XML em duas partes principais: a de metadados, que dá a definição das variáveis que serão utilizadas, e a dos dados propriamente ditos, onde estarão depositados as informações do estudo clínico.

![O esquema de um documento XML para dados clínicos, segundo o modelo ODM estabelecido pelo CDISC. <sup id="6">[6](#seis)</sup>](https://www.researchgate.net/profile/Hugo-Leroux/publication/259220016/figure/fig1/AS:297314913669122@1447896807509/Structure-of-CDISC-ODM-XML-schema-adapted-from-ODM-11-documentation-9.png '')

![Outro esquema de um documento XML para dados clínicos. <sup id="7">[7](#sete)</sup>](https://storage.googleapis.com/plos-corpus-prod/10.1371/journal.pone.0199242/1/pone.0199242.g001.PNG_L?X-Goog-Algorithm=GOOG4-RSA-SHA256&X-Goog-Credential=wombat-sa%40plos-prod.iam.gserviceaccount.com%2F20240806%2Fauto%2Fstorage%2Fgoog4_request&X-Goog-Date=20240806T163904Z&X-Goog-Expires=86400&X-Goog-SignedHeaders=host&X-Goog-Signature=3a72e746188a570c170ed250887b92b6333cf74bae5bfa55881833e45598e49d956e27b7e3feac39ec130c6230eb86bdcad4ae32040aa05c0b446ab00c992b7bf39011aacd7e6c7d3bcb907b28de3c66373c0180dce00d0b3a4cc766a65379f3e2054ffaa9ea9d10652029e0318d4a9c93fefa94c64c3f0b560f57e665926fbefb94191bfec9a0be0e0fc0067b584cac4c4000a4f61c0b190970bd88e19fcc78bc3ba4e56118365ded053713e0fb619e07c222a79a9e29a34d5420df2aa183f65ab6cce2b5ea5f990d4fd96ffc8ba00b2afdc627d08785fca598f7ee2b50fac469464b22d81aab2df9b0890bf72b82c355f128d086f0c4af06900771dc49ae3a, '')

## Importando bibliotecas

Ao usar o Python para extrair dados de documentos `.xml`, é preciso importar algumas bibliotecas; se você usa plataformas como o [Anaconda](https://www.anaconda.com/) ou o [WinPython](https://winpython.github.io/), é provável que apenas o uso do `import` já seja o suficiente. Se você estiver usando uma versão "pura" do Python, recomendo que, antes de executar os passos a seguir, seja feita a instalação das bibliotecas usando o [`pip`](https://pt.wikipedia.org/wiki/Pip_(gerenciador_de_pacotes)).

Neste artigo, usaremos bibliotecas como:

- [`requests`](https://requests.readthedocs.io/en/latest/), a biblioteca mais simples para requisições envolvendo páginas web (porque de complexa já basta a vida);
- [`lxml`](https://lxml.de/), outra biblioteca 'easy-to-use', esta para processar os arquivos XML;
- [`pandas`](https://pandas.pydata.org/), que é a biblioteca mais conhecida para manipulação e análise de dados; com ela, será possível construir uma planilha com os dados do XML.

```python
import requests
from lxml import etree #Neste caso, usaremos a API da biblioteca 'ElementTree' que está disponível na biblioteca 'lxml'
import pandas as pd
```

## Obtendo os dados

O documento `.xml` que será utilizado origina-se de um [repositório no GitHub da CDISC](https://github.com/cdisc-org/DataExchange-ODM); para obtê-lo diretamente, i.e., sem precisar fazer o download de qualquer arquivo, usaremos a biblioteca `requests`. A função [`get`](https://www.w3schools.com/python/ref_requests_get.asp) dessa biblioteca faz uma requisição a uma página do GitHub que contém o arquivo, e espera uma resposta dessa página em forma de código, que estamos armazenando na variável `response`. Se a resposta for o código `200`, [significa que a requisição foi bem-sucedida](https://www.geeksforgeeks.org/response-methods-python-requests/).

```python
url = 'https://github.com/cdisc-org/DataExchange-ODM/raw/main/examples/Demographics_RACE/Demographics_RACE_check_all_that_apply.xml'
response = requests.get(url)
print(response)
```

```txt
<Response [200]>
```

A partir dessa resposta, utilizaremos o objeto [`content`](https://www.geeksforgeeks.org/response-content-python-requests/) para de fato obter o XML que será explorado. O conteúdo estará disponível na variável `tree`, conforme o código que está abaixo:

```python
tree = response.content
```

A variável `tree` nos acompanhará durante todo o processo de construção da planilha a partir das informações que temos.

## Fazendo a transformação e verificando a estrutura do XML

Feita a extração do conteúdo que está na página do GitHub, é possível observar a estrutura do XML que será explorado, quando o conteúdo será transformado em uma variável legível; isso é importante para serem localizadas as tags, atributos, e valores onde estão os dados que interessam de fato.

A partir daqui, começamos o uso de outra biblioteca que importamos: `lxml`, que se dedicará à obtenção dos elementos XML que já foram mencionados. Armazenaremos o XML inteiro na variável `tree`, onde serão passados o conteúdo da página do GitHub, além do tipo de parser (o transformador) que será utilizado. A variável `tree2`, neste caso, serve para podermos observar de fato o "esqueleto" do XML.

```python
tree = etree.XML(tree, etree.XMLParser(remove_comments=True))
tree2 = etree.tostring(tree, pretty_print = True, encoding = str)
print(tree2)
```

```txt
    <ODM xmlns="http://www.cdisc.org/ns/odm/v2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xlink="http://www.w3.org/1999/xlink" CreationDateTime="2020-07-06T10:20:15+01:00" FileOID="DEMOGRAPHICS_EXAMPLE" FileType="Snapshot" Granularity="Metadata" ODMVersion="2.0" SourceSystem="XML4Pharma CDISC ODM Study Designer" SourceSystemVersion="2015-R1">
       <Study OID="ST.DEMOGRAPHICS_EXAMPLE" StudyName="Study with Demographics example" ProtocolName="MyStudy">
          <Description><TranslatedText xml:lang="en" Type="text/plain">Demographics example with Race, with "check all that apply"</TranslatedText></Description>
          <MetaDataVersion Name="Version 1" OID="MV.1.0">
             <Description><TranslatedText xml:lang="en" Type="text/plain">Version 1</TranslatedText></Description>
             <StudyEventDef Name="Screening visit with demographics" OID="SE.SCREENING" Repeating="No" Type="Scheduled">
                <ItemGroupRef ItemGroupOID="FO.DEMOGRAPHICS" Mandatory="Yes"/>
             </StudyEventDef>
             <ItemGroupDef Name="Demographics form" OID="FO.DEMOGRAPHICS" Type="Form" Repeating="No">
                <ItemGroupRef ItemGroupOID="IG.DEMOGRAPHICS" Mandatory="Yes"/>
             </ItemGroupDef>
             <ItemGroupDef Name="Demographics" OID="IG.DEMOGRAPHICS" Type="Section" Repeating="No">
                <ItemRef ItemOID="IT.DOB" Mandatory="Yes"/>
                <ItemRef ItemOID="IT.SEX" Mandatory="Yes"/>
                <ItemRef ItemOID="IT.ETHNIC" Mandatory="Yes"/>
                
                
                <ItemGroupRef ItemGroupOID="IG.RACE" Mandatory="Yes"/>
             </ItemGroupDef>
             
             ...

             <CodeList DataType="integer" Name="Sex" OID="CL.SEX">
                <CodeListItem CodedValue="1">
                   <Decode>
                      <TranslatedText xml:lang="en" Type="text/plain">Male</TranslatedText>
                   </Decode>
                </CodeListItem>
                <CodeListItem CodedValue="2">
                   <Decode>
                      <TranslatedText xml:lang="en" Type="text/plain">Female</TranslatedText>
                   </Decode>
                </CodeListItem>
             </CodeList>
             <CodeList DataType="integer" Name="Ethnicity" OID="CL.ETHNIC">
                <CodeListItem CodedValue="1">
                   <Decode>
                      <TranslatedText xml:lang="en" Type="text/plain">Hispanic</TranslatedText>
                   </Decode>
                </CodeListItem>
                <CodeListItem CodedValue="2">
                   <Decode>
                      <TranslatedText xml:lang="en" Type="text/plain">Non-hispanic</TranslatedText>
                   </Decode>
                </CodeListItem>
             </CodeList>

             ...

          </MetaDataVersion>
       </Study>
       
       <ClinicalData StudyOID="ST.DEMOGRAPHICS_EXAMPLE" MetaDataVersionOID="MV.1.0">
          
          <SubjectData SubjectKey="001">
             <StudyEventData StudyEventOID="SE.SCREENING">
                <ItemGroupData ItemGroupOID="FO.DEMOGRAPHICS">
                   <ItemGroupData ItemGroupOID="IG.DEMOGRAPHICS">
                      <ItemData ItemOID="IT.DOB"><Value>1957-05-07</Value></ItemData>
                      <ItemData ItemOID="IT.SEX"><Value>1</Value></ItemData>
                      <ItemData ItemOID="IT.ETHNIC"><Value>2</Value></ItemData>
                      
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="1">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>1</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="2">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>2</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>true</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="3">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>3</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="4">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>4</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>4</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="5">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>5</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="6">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>99</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                   </ItemGroupData>
                </ItemGroupData>
             </StudyEventData>
          </SubjectData>
          
          ...

          <SubjectData SubjectKey="003">
             <StudyEventData StudyEventOID="SE.SCREENING">
                <ItemGroupData ItemGroupOID="FO.DEMOGRAPHICS">
                   <ItemGroupData ItemGroupOID="IG.DEMOGRAPHICS">
                      <ItemData ItemOID="IT.DOB"><Value>1961-06-09</Value></ItemData>
                      <ItemData ItemOID="IT.SEX"><Value>2</Value></ItemData>
                      <ItemData ItemOID="IT.ETHNIC"><Value>1</Value></ItemData>
                      
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="1">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>1</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="2">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>2</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="3">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>3</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="4">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>4</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="5">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>5</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="6">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>99</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>true</Value></ItemData>
                         <ItemData ItemOID="IT.RACEOTH"><Value>Native Amazonian</Value></ItemData>
                      </ItemGroupData>
                   </ItemGroupData>
                </ItemGroupData>
             </StudyEventData>
          </SubjectData>
       </ClinicalData>
    </ODM>
```

## Verificando os namespaces

Ao explorarmos um XML, é essencial que tomemos conhecimento dos [namespaces](https://en.wikipedia.org/wiki/XML_namespace) que este possui. Os namespaces podem servir de identificação única para cada tag componente. O uso de namespaces é recomendado pela W3C, e se vê muito necessário quando temos tags ou atributos de nome semelhante, mas associado a elementos ou tags diferentes. Para vermos quais namespaces aparecem no documento XML, utliza-se o objeto [`nsmap`](https://lxml.de/4.3/api/lxml.etree._Element-class.html), como abaixo:

```python
ns = tree.nsmap
print(ns)
```

```txt
{None: 'http://www.cdisc.org/ns/odm/v2.0', 'xs': 'http://www.w3.org/2001/XMLSchema', 'xlink': 'http://www.w3.org/1999/xlink'}
```

## Verificando as tags

Tendo já conhecimento dos namespaces, podemos agora analisar quais são os nomes daquilo que chamamos de *tags*. É através desses nomes que será possível extrair os dados de interesse mais para frente.

```python
elements = []
for elem in tree.iter():
    elements.append(elem.tag)
    
elements = list(set(elements))
print(elements)
```

```txt
['{http://www.cdisc.org/ns/odm/v2.0}TranslatedText', '{http://www.cdisc.org/ns/odm/v2.0}Description', '{http://www.cdisc.org/ns/odm/v2.0}ItemGroupDef', '{http://www.cdisc.org/ns/odm/v2.0}Decode', '{http://www.cdisc.org/ns/odm/v2.0}ClinicalData', '{http://www.cdisc.org/ns/odm/v2.0}ODM', '{http://www.cdisc.org/ns/odm/v2.0}ItemGroupData', '{http://www.cdisc.org/ns/odm/v2.0}ItemRef', '{http://www.cdisc.org/ns/odm/v2.0}Question', '{http://www.cdisc.org/ns/odm/v2.0}CodeListItem', '{http://www.cdisc.org/ns/odm/v2.0}SubjectData', '{http://www.cdisc.org/ns/odm/v2.0}ItemGroupRef', '{http://www.cdisc.org/ns/odm/v2.0}MetaDataVersion', '{http://www.cdisc.org/ns/odm/v2.0}StudyEventData', '{http://www.cdisc.org/ns/odm/v2.0}Study', '{http://www.cdisc.org/ns/odm/v2.0}Value', '{http://www.cdisc.org/ns/odm/v2.0}CodeListRef', '{http://www.cdisc.org/ns/odm/v2.0}StudyEventDef', '{http://www.cdisc.org/ns/odm/v2.0}ItemDef', '{http://www.cdisc.org/ns/odm/v2.0}ItemData', '{http://www.cdisc.org/ns/odm/v2.0}CodeList']
```

## Extraindo os primeiros atributos

Sabendo já como o arquivo XML está estruturado, bem como quais são as tags que estão presentes, é possível selecionar os locais onde estão os dados de interesse. A próxima coisa a se fazer é analisar o que está dentro das tags. A esse conteúdo damos o nome de atributos. Atributos são partes internas das tags que obedecem um padrão `'Nome="Valor"'`. Perceba o atributo `'ItemOID'` no final da primeira linha; ele vem acompanhado de um valor `"IT.DOB"`. Este é o padrão utilizado e recomendado pela W3C quando se tratam de atributos. <sup id="8">[8](#oito)</sup>

```python
print(etree.tostring(tree.find('.//ItemData', ns), pretty_print = True, encoding = str))
```

```txt
    <ItemData xmlns="http://www.cdisc.org/ns/odm/v2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xlink="http://www.w3.org/1999/xlink" ItemOID="IT.DOB">
      <Value>1957-05-07</Value>
    </ItemData>
```

Para acessar os atributos de uma tag em específico, é possível utilizar duas funções do `lxml`, dependendo da situação: [`find`](https://docs.python.org/3/library/xml.etree.elementtree.html#xml.etree.ElementTree.Element.find) e [`findall`](https://docs.python.org/3/library/xml.etree.elementtree.html#xml.etree.ElementTree.Element.findall). A primeira serve para resgatar somente a primeira ocorrência de uma tag no arquivo XML, enquanto que a outra retorna todas as ocorrências de tags com o nome mencionado.  

```python
tree.find('.//ClinicalData', ns) #Apenas a primeira ocorrência da tag ClinicalData
```

```txt
<Element {http://www.cdisc.org/ns/odm/v2.0}ClinicalData at 0x21052103e80>
```

```python
tree.findall('.//SubjectData', ns) #Todas as ocorrências da tag SubjectData
```

```txt
[<Element {http://www.cdisc.org/ns/odm/v2.0}SubjectData at 0x21052102e00>,
<Element {http://www.cdisc.org/ns/odm/v2.0}SubjectData at 0x21052108540>,
<Element {http://www.cdisc.org/ns/odm/v2.0}SubjectData at 0x21052108840>]
```

Perceba que, ao usarmos estas funções, o retorno é um objeto de classe [`Element`](https://docs.python.org/3/library/xml.etree.elementtree.html#element-objects); contudo, não é exatamente isso que está sendo procurado. Para buscar o que realmente são as tags e os atributos dentro delas, podemos nos valer de dois objetos, cujos nomes são sugestivos: `attrib` e `tag`. No caso do atributo, os resultados retornam em forma de dicionário, onde a chave é o nome do atributo, e o valor é o valor desse mesmo atributo. A função `tag`, por sua vez, retorna os nomes das tags que foram mencionadas.

```python
tree.find('.//ClinicalData', ns).attrib #Atributos apenas da primeira (e única) tag ClinicalData
```

```txt
{'StudyOID': 'ST.DEMOGRAPHICS_EXAMPLE', 'MetaDataVersionOID': 'MV.1.0'}
```

```python
for subject in tree.findall('.//SubjectData', ns):
    subj = subject.attrib #Atributos de cada tag SubjectData
    print(subj)
```

```txt
{'SubjectKey': '001'}
{'SubjectKey': '002'}
{'SubjectKey': '003'}
```

```python
tree.find('.//SubjectData', ns).tag #Nome da tag SubjectData
```

```txt
'{http://www.cdisc.org/ns/odm/v2.0}SubjectData'
```

```python
for subject in tree.findall('.//SubjectData', ns):
    for eve in subject:
        print(eve.attrib) #Atributos das tags StudyEventData
```

```txt
{'StudyEventOID': 'SE.SCREENING'}
{'StudyEventOID': 'SE.SCREENING'}
{'StudyEventOID': 'SE.SCREENING'}
```

```python
for subject in tree.findall('.//SubjectData', ns):
    for event in subject:
        for item in event:
            ite = item.attrib
            print(ite) #Atributos da tag ItemGroupDef
```

```txt
{'ItemGroupOID': 'FO.DEMOGRAPHICS'}
{'ItemGroupOID': 'FO.DEMOGRAPHICS'}
{'ItemGroupOID': 'FO.DEMOGRAPHICS'}
```

Caso se queira extrair somente o valor de um determinado atributo, basta fazer uma seleção do atributo com seu nome em colchetes, o mesmo que se faz quando se quer descobrir o valor de uma determinada chave armazenada em um dicionário:

```python
print(subj['SubjectKey']) #Valor do atributo SubjectKey na última tag SubjectData
```

```txt
003
```

## Verificando a estrutura do XML para apenas um indivíduo

Nós já vimos como fazer uma filtragem de uma ou mais tags através de seus nomes, e descobrir os atributos e seus valores. Caso seja interessante ou necessário analisar mais a fundo a estrutura XML de apenas uma ocorrência de tag (nesse caso, um indivíduo), basta utilizar a função `find`, que já conhecemos. Com o código abaixo, podemos verificar a estrutura do XML para o primeiro indivíduo.

```python
print(etree.tostring(tree.find('.//SubjectData', ns), pretty_print = True, encoding = str))
```

```txt
    <SubjectData xmlns="http://www.cdisc.org/ns/odm/v2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xlink="http://www.w3.org/1999/xlink" SubjectKey="001">
             <StudyEventData StudyEventOID="SE.SCREENING">
                <ItemGroupData ItemGroupOID="FO.DEMOGRAPHICS">
                   <ItemGroupData ItemGroupOID="IG.DEMOGRAPHICS">
                      <ItemData ItemOID="IT.DOB"><Value>1957-05-07</Value></ItemData>
                      <ItemData ItemOID="IT.SEX"><Value>1</Value></ItemData>
                      <ItemData ItemOID="IT.ETHNIC"><Value>2</Value></ItemData>
                      
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="1">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>1</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="2">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>2</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>true</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="3">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>3</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="4">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>4</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>4</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="5">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>5</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="6">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>99</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                   </ItemGroupData>
                </ItemGroupData>
             </StudyEventData>
          </SubjectData>
          
```

Agora, se queremos filtrar as tags e atributos referentes a um outro indivíduo em específico, é preciso mencionar o atributo `SubjectKey` e mencionar o valor do indivíduo (no caso, '002'), como o código abaixo:

```python
print(etree.tostring(tree.find('.//SubjectData[@SubjectKey="002"]', ns), pretty_print = True, encoding = str))
```

```txt
    <SubjectData xmlns="http://www.cdisc.org/ns/odm/v2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xlink="http://www.w3.org/1999/xlink" SubjectKey="002">
             <StudyEventData StudyEventOID="SE.SCREENING">
                <ItemGroupData ItemGroupOID="FO.DEMOGRAPHICS">
                   <ItemGroupData ItemGroupOID="IG.DEMOGRAPHICS">
                      <ItemData ItemOID="IT.DOB"><Value>1975-01-31&gt;</Value></ItemData>
                      <ItemData ItemOID="IT.SEX"><Value>2</Value></ItemData>
                      <ItemData ItemOID="IT.ETHNIC"><Value>2</Value></ItemData>
                      
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="1">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>1</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>1</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="2">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>2</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="3">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>3</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>true</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="4">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>4</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="5">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>5</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                      <ItemGroupData ItemGroupOID="IG.RACE" ItemGroupRepeatKey="6">
                         <ItemData ItemOID="IT.RACE_CODE"><Value>99</Value></ItemData>
                         <ItemData ItemOID="IT.RACE_BOOLEAN"><Value>false</Value></ItemData>
                      </ItemGroupData>
                   </ItemGroupData>
                </ItemGroupData>
             </StudyEventData>
          </SubjectData>
          
```

## Buscando dados de uma tag apenas

Assim como podemos analisar como um documento XML inteiro está estruturado, é possível especificar uma tag e separar sua estrutura, podendo assim analisar como está disposta a prórpia tag, bem como as tags que estão ligadas a ela.

```python
print(etree.tostring(tree.find('.//ItemData', ns), pretty_print = True, encoding = str))
```

```txt
    <ItemData xmlns="http://www.cdisc.org/ns/odm/v2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xlink="http://www.w3.org/1999/xlink" ItemOID="IT.DOB">
      <Value>1957-05-07</Value>
    </ItemData>
```

Agora, verifique a tag `Value`: ela possui um valor que nos interessa extrair, mas não está em um atributo; é um texto que está entre as tags. Para acessar o conteúdo que está entre as tags de abertura e fechamento, podemos nos valer da classe `text` do `lxml`.

```python
tree.find('.//Value', ns).text
```

```txt
'1957-05-07'
```

O mesmo podemos fazer com o nome da tag, usando dessa vez a classe `tag`.

```python
tree.find('.//Value', ns).tag
```

```txt
'{http://www.cdisc.org/ns/odm/v2.0}Value'
```

## Produzindo planilhas

Sabendo já alguns pontos básicos de XML, e como manipulá-los usando bibliotecas do Python, podemos agora ir para a fase seguinte: produzir planilhas (ou produzir DataFrames). Mas antes de fazermos isso, com o que já vimos, organizaremos um esquema de extração de dados usando o `lxml`.

O nosso objetivo aqui é elaborar um DataFrame que contenha os dados de *data de nascimento*, *sexo*, e *etnia* de cada indivíduo registrado no documento XML. Para tanto, adotaremos a estratégia de estabelecer um loop em `for`, para que, a cada iteração, consigamos extrair os dados associados à tag `SubjectData` e suas herdeiras. Feitas as iterações, armazenaremos os dados em uma lista `results`, cujo resultado aparece abaixo. É um método de resultado interessante, dado que os valores extraídos estão organizados em três grupos próprios, que serão as colunas do nosso futuro DataFrame.

```python
results = []
for ide in tree.findall('.//SubjectData', ns):
    for subj in ide.findall('.//StudyEventData/ItemGroupData/ItemGroupData/ItemData', ns):
            for value in subj:
                results.append([ide.attrib['SubjectKey'], subj.attrib['ItemOID'], value.text])
                
results
```

```txt
[['001', 'IT.DOB', '1957-05-07'],
['001', 'IT.SEX', '1'],
['001', 'IT.ETHNIC', '2'],
['002', 'IT.DOB', '1975-01-31>'],
['002', 'IT.SEX', '2'],
['002', 'IT.ETHNIC', '2'],
['003', 'IT.DOB', '1961-06-09'],
['003', 'IT.SEX', '2'],
['003', 'IT.ETHNIC', '1']]
```

Elaborada a lista e armazenada na variável `results`, podemos agora facilmente usar o `pandas` para fazer um DataFrame, com a função `pd.DataFrame`. É fácil pois podemos utilizar a lista que produzimos diretamente, sem precisar de mais transformações. Nessa função, passaremos a lista como um argumento, e faremos menção aos nomes das colunas que queremos que apareça. São elas `ID`, `Variable`, `Value`. É possível, sem quaisquer problemas, colocar qualquer nome a cada coluna.

```python
results = pd.DataFrame(results, columns=['ID', 'Variable', 'Value']) 
results
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ID</th>
      <th>Variable</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>001</td>
      <td>IT.DOB</td>
      <td>1957-05-07</td>
    </tr>
    <tr>
      <th>1</th>
      <td>001</td>
      <td>IT.SEX</td>
      <td>1</td>
    </tr>
    <tr>
      <th>2</th>
      <td>001</td>
      <td>IT.ETHNIC</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>002</td>
      <td>IT.DOB</td>
      <td>1975-01-31&gt;</td>
    </tr>
    <tr>
      <th>4</th>
      <td>002</td>
      <td>IT.SEX</td>
      <td>2</td>
    </tr>
    <tr>
      <th>5</th>
      <td>002</td>
      <td>IT.ETHNIC</td>
      <td>2</td>
    </tr>
    <tr>
      <th>6</th>
      <td>003</td>
      <td>IT.DOB</td>
      <td>1961-06-09</td>
    </tr>
    <tr>
      <th>7</th>
      <td>003</td>
      <td>IT.SEX</td>
      <td>2</td>
    </tr>
    <tr>
      <th>8</th>
      <td>003</td>
      <td>IT.ETHNIC</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>

Conseguimos um DataFrame, mas precisamos ir um pouco mais além antes de dar tudo como terminado. A ideia principal aqui é fazer com que `results` esteja com apenas um ID por linha, fazendo com que todos os dados referentes ao ID estejam nessa mesma linha. Aqui vemos que poderíamos utilizar os valores da coluna `Variable` como nomes das colunas, e o que está na coluna `Value` seriam os valores de cada coluna por ID. 

A boa notícia é que o `pandas` permite fazer isso sem o menor problema. O que iremos performar agora é uma *pivotagem* dos dados que temos. Para tanto, podemos utilizar duas funções: `pivot` e `pivot_table`, com uma leve diferença entre as duas. A função escolhida aqui é a `pivot_table`, onde vamos lançar o `results` como fonte de dados, a coluns `ID` como índice temporário do DataFrame, a coluna `Value` como quem dará o nome às novas colunas, e a coluna `Value` como quem dará os valores às colunas. Ainda, teremos de invocar uma função no argumento `aggfunc`, que serve como uma função para fazer cálculos ou lançar um dado de forma ordenada (o primeiro dado a aparecer, o último, etc.). Nesse caso, usaremos a função `first`, porque simplesmente queremos que o primeiro (e único) valor a aparecer seja aquele aparente no DataFrame. Para fechar, utilizaremos o `reset_index`, fazendo com que `ID` volte a ser uma coluna manipulável.

```python
results = pd.pivot_table(results, index='ID', columns='Variable', values='Value', aggfunc='first').reset_index()
results
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>Variable</th>
      <th>ID</th>
      <th>IT.DOB</th>
      <th>IT.ETHNIC</th>
      <th>IT.SEX</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>001</td>
      <td>1957-05-07</td>
      <td>2</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>002</td>
      <td>1975-01-31&gt;</td>
      <td>2</td>
      <td>2</td>
    </tr>
    <tr>
      <th>2</th>
      <td>003</td>
      <td>1961-06-09</td>
      <td>1</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>

Assim estamos dando uma aparência mais definitiva ao nosso DataFrame. Mas veja que os nomes das colunas, bem como os valores das últimas duas planilhas ainda não nos informam com clareza qual a informação a ser mostrada. Analisando mais atentamente ao documento XML, se percebe que há uma parte nele que nos fornece informações valiosas para darmos mais sentido aos dados que temos: a parte dos *metadados*. É isso que exploraremos a partir de agora.

## Buscando os metadados do documento XML (e melhorando a planilha)

Os metadados são as definições de cada variável e valor dentro do documento XML. É interessante, até preciso, obtê-los para que possamos compreender de fato o que cada coluna e valor representa em um DataFrame. Neste documento em específico, os metadados que nos interessam estão depositados nas tags de nome `ItemDef`, e podemos extraí-los com o bloco de código ilustrado abaixo:

```python
for meta in tree.findall('.//ItemDef', ns):
    print(meta.attrib)
```

```txt
{'DataType': 'date', 'Name': 'Date of birth', 'OID': 'IT.DOB'}
{'DataType': 'integer', 'Length': '1', 'Name': 'Sex', 'OID': 'IT.SEX'}
{'DataType': 'integer', 'Length': '1', 'Name': 'Ethnicity', 'OID': 'IT.ETHNIC'}
{'OID': 'IT.RACE_CODE', 'Name': 'Race code', 'DataType': 'integer', 'Length': '1'}
{'DataType': 'boolean', 'Length': '1', 'Name': 'Race', 'OID': 'IT.RACE_BOOLEAN'}
{'DataType': 'text', 'Length': '20', 'Name': 'Other Race', 'OID': 'IT.RACEOTH'}
```

Perceba que o retorno é uma série de dicionários cujas chaves indicam aspectos como 'Nome' e 'OID' (um ID único para cada objeto que compõe o XML); para extrair esses dois e torná-los úteis para convertermos os nomes em código das colunas em um nome que nos indica o que determinada coluna realmente representa. Para tanto, criaremos um dicionário com o OID de cada coluna como chave, e o valor como o nome da coluna, como está abaixo:

```python
names = {}
for meta in tree.findall('.//ItemDef', ns):
    names[meta.attrib['OID']] = meta.attrib['Name']
names
```

```txt
{'IT.DOB': 'Date of birth',
'IT.SEX': 'Sex',
'IT.ETHNIC': 'Ethnicity',
'IT.RACE_CODE': 'Race code',
'IT.RACE_BOOLEAN': 'Race',
'IT.RACEOTH': 'Other Race'}
```

Agora, com a função `rename` do pandas, podemos mudar os nomes das colunas em `results`, e as coisas começam a ter mais sentido.

```python
results = results.rename(columns=names)
results
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>Variable</th>
      <th>ID</th>
      <th>Date of birth</th>
      <th>Ethnicity</th>
      <th>Sex</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>001</td>
      <td>1957-05-07</td>
      <td>2</td>
      <td>1</td>
    </tr>
    <tr>
      <th>1</th>
      <td>002</td>
      <td>1975-01-31&gt;</td>
      <td>2</td>
      <td>2</td>
    </tr>
    <tr>
      <th>2</th>
      <td>003</td>
      <td>1961-06-09</td>
      <td>1</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>

Nós temos aqui também duas colunas que estão em códigos numéricos: `Ethnicity` e `Sex`. As definições desses códigos, costumeiramente, estão no início do documento XML, junto dos metadados. Para obtermos os nomes dos códigos de sexo e etnia, nesse caso, há uma tag que será nosso alvo: `CodeList`. O código abaixo mostra como a tag está estruturada:

```python
print(etree.tostring(tree.find('.//CodeList', ns), pretty_print = True, encoding = str))
```

```txt
    <CodeList xmlns="http://www.cdisc.org/ns/odm/v2.0" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:xlink="http://www.w3.org/1999/xlink" DataType="integer" Name="Sex" OID="CL.SEX">
                <CodeListItem CodedValue="1">
                   <Decode>
                      <TranslatedText xml:lang="en" Type="text/plain">Male</TranslatedText>
                   </Decode>
                </CodeListItem>
                <CodeListItem CodedValue="2">
                   <Decode>
                      <TranslatedText xml:lang="en" Type="text/plain">Female</TranslatedText>
                   </Decode>
                </CodeListItem>
             </CodeList>
```

Agora, dê uma recordada do método anterior que utilizamos para obter os nomes de colunas. Basicamente, o princípio para obter os valores dos códigos numéricos é o mesmo; contudo, ele é um pouco mais complexo, demandando um pouco mais de linhas de código. Ao acessar a tag `CodeList`, para o caso de querermos extrair os valores para `Ethnicity`, precisamos fazer uma filtragem mencionando o nome desse atributo, através do XPATH. Isso vem logo depois do nome da tag; assim sendo, podemos criar um novo dicionário, `ethnicity`, para abrigar códigos e valores para as etnias que estão representadas no DataFrame.

```python
ethnicity = {}
for eth in tree.findall('.//CodeList[@Name="Ethnicity"]', ns):
    for code in tree.findall('.//CodeList[@Name="Ethnicity"]/CodeListItem', ns):
        for decode in code:
            for name in decode:
                ethnicity[code.attrib['CodedValue']] = name.text
ethnicity
```

```txt
{'1': 'Hispanic', '2': 'Non-hispanic'}
```

Para obtermos os valores dos códigos da coluna `Sex`, basta usar o mesmo método anterior, apenas substituindo o nome do atributo a ser filtrado.

```python
sex = {}
for eth in tree.findall('.//CodeList[@Name="Sex"]', ns):
    for code in tree.findall('.//CodeList[@Name="Sex"]/CodeListItem', ns):
        for decode in code:
            for name in decode:
                sex[code.attrib['CodedValue']] = name.text
sex
```

```txt
{'1': 'Male', '2': 'Female'}
```

Dicionários preparados, podemos proceder às substituições que faltam. Aqui, usaremos a função do pandas `map`, que associa as chaves (i.e., os códigos numéricos) dos dicionários que criamos aos valores que dão nome aos números. Faremos isso para as duas colunas, `Ethnicity` e `Sex`; perceba que a planilha agora faz muito mais sentido, e está já pronta para fazer as análises que se deseja.

```python
results['Ethnicity'] = results['Ethnicity'].map(ethnicity)
results['Sex'] = results['Sex'].map(sex)
results
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th>Variable</th>
      <th>ID</th>
      <th>Date of birth</th>
      <th>Ethnicity</th>
      <th>Sex</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>001</td>
      <td>1957-05-07</td>
      <td>Non-hispanic</td>
      <td>Male</td>
    </tr>
    <tr>
      <th>1</th>
      <td>002</td>
      <td>1975-01-31&gt;</td>
      <td>Non-hispanic</td>
      <td>Female</td>
    </tr>
    <tr>
      <th>2</th>
      <td>003</td>
      <td>1961-06-09</td>
      <td>Hispanic</td>
      <td>Female</td>
    </tr>
  </tbody>
</table>
</div>

Para salvar o DataFrame que foi criado, por fim, basta usar as funções próprias do `pandas` para tanto. Assim as informações que foram extraídas estarão armazenadas em um arquivo mais leve e mais rápido de manipular.

```python
## Exportando

results.to_csv('results.csv') #Arquivo .csv
results.to_excel('results.xlsx') #Arquivo .xlsx
results.to_parquet('results.parquet') #Arquivo .parquet
```

## Notas do artigo

---

<span id='um'></span> 1. Isso é sucintamente e muito bem corroborado em Shabo et. al (2006): “The Clinical Data Interchange Standards Consortium (CDISC) is leading the development of standards to improve data quality and accelerate product development in the pharmaceutical industry.19 The CDISC model focuses on the use of metadata, and the approach is to combine XML representation with the tabular presentation traditionally used for clinical-trial data.” ([Shabo et al., 2006, p. 3](zotero://select/library/items/QGS52CN8)) ([pdf](zotero://open-pdf/library/items/GNJ7QELP?page=4&annotation=3HT5764G)) [↩](#1)

<span id='dois'></span> 2. “Interest in ODM as a research topic has grown significantly over the last several years with increasing interest in the CDISC data standards from regulatory authorities such as the FDA and the Japanese Pharmaceutical and Medical Devices Agency (PMDA)” ([Hume et al., 2016, p. 3](zotero://select/library/items/VXGWBTWS)) ([pdf](zotero://open-pdf/library/items/N3A38ELP?page=3&annotation=KJW2ZPB7)) [↩](#2)

<span id='três'></span> 3. “While it is a requirement to submit pre-clinical and clinical data in CDISC format to regulatory bodies such as the US FDA and Japan’s Pharmaceuticals and Medical Devices Agency (PDMA), the actual usage of CDISC standards spans a much wider array of entities.” ([Hufstedler et al., p. 4](zotero://select/library/items/ADWTACAD)) ([pdf](zotero://open-pdf/library/items/VCYXKF4I?page=4&annotation=9M9XDCDM)) [↩](#3)

<span id='quatro'></span> 4. “The Federal Drug Administration has mandated the use of the CDISC standards for the electronic capture and reporting of clinical study data” ([Leroux et al., 2017, p. 1](zotero://select/library/items/LGMJ3MVC)) ([pdf](zotero://open-pdf/library/items/YCJ3HXDD?page=1&annotation=6Y7V8BPA)) [↩](#4)

<span id='cinco'></span> 5. “The Federal Drug Administration has mandated the use of the CDISC standards for the electronic capture and reporting of clinical study data” ([Leroux et al., 2017, p. 1](zotero://select/library/items/LGMJ3MVC)) ([pdf](zotero://open-pdf/library/items/YCJ3HXDD?page=1&annotation=6Y7V8BPA)) [↩](#5)

<span id='seis'></span> 6. Lefort e Leroux, “Design and generation of Linked Clinical Data Cubes”. 2013. [↩](#6)

<span id='sete'></span> 7. Brix et al., “ODM Data Analysis—A Tool for the Automatic Validation, Monitoring and Generation of Generic Descriptive Statistics of Patient Data”. 2018. [↩](#7)

<span id='oito'></span> 8. **Uma nota importante**: A partir de agora, além das bibliotecas Python, e do XML, nos valeremos de uma outra linguagem, esta de consulta: o *XPath*. Com ela, podemos acessar de forma apropriada os elementos e atributos do XML. Não entrarei em detalhes sobre ela neste artigo; mas, caso você queira entender melhor do que se trata, você pode ver mais detalhes [aqui](https://escoladedados.org/tutoriais/xpath-para-raspagem-de-dados-em-html/) e [aqui](https://www.w3schools.com/xml/xpath_intro.asp), além de um bom cheatsheet [aqui](https://devhints.io/xpath). Não usaremos muitas coisas diferentes dessa linguagem por aqui, mas é interessante ir mais a fundo depois. [↩](#8)

---
