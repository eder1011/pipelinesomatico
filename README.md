# Pipeline Somático
Pipeline Somático - Do VCF (anotado) até o CGI Classificação

**1. Clonar o repositório GitHub lmabrasil-hg38**

***Baixa para o ambiente do Google Colab todo o conteúdo do repositório GitHub.***

***Isso permite acessar arquivos necessários para preparar os dados que serão enviados à API do Cancer Genome Interpreter (CGI).***


```bash
! git clone https://github.com/renatopuga/lmabrasil-hg38.git
```
**output:** 

```
Cloning into 'lmabrasil-hg38'...
remote: Enumerating objects: 226, done.
remote: Counting objects: 100% (168/168), done.
remote: Compressing objects: 100% (108/108), done.
remote: Total 226 (delta 90), reused 114 (delta 56), pack-reused 58 (from 1)
Receiving objects: 100% (226/226), 8.63 MiB | 32.38 MiB/s, done.
Resolving deltas: 100% (106/106), done.
```

**2. Preparar arquivo para envio ao CGI (API REST)**

***cut -f1-4 -> Extrai apenas as colunas 1 a 4 do arquivo VEP.***

***Essas colunas representam: CHROM, POS, REF e ALT.***

***sed -e "s/CHROM/CHR/g" -> Substitui o nome da coluna CHROM por CHR, o CGI exige esse formato no arquivo de entrada.***

***> df_WP048-cgi.txt -> Cria um novo arquivo somente com as informações necessárias para a API.***

***head -> Lista as primeiras 10 linhas do arquivo para conferência.***

```bash
%%bash
#cortar pelas colunas de 1 a 4 e criar um novo arquivo chamado df_WP048-cgi.txt
#1: CHROM (converte CHROM para CHR) formato que CGI gosta
#2: POS
#3: REF
#4: ALT
cut -f1-4 /content/lmabrasil-hg38/vep_output/liftOver_WP048_hg19ToHg38.vep.filter.tsv | sed -e "s/CHROM/CHR/g"  > df_WP048-cgi.txt
head df_WP048-cgi.txt

#listar as 10 primeiras linhas
head df_WP048-cgi.txt
```

**output:** 

```
CHR	POS	REF	ALT
chr1	114716123	C	T
chr9	5073770	G	T
```

**3. Enviar Job para o CGI (Cancer Genome Interpreter).**
>https://www.cancergenomeinterpreter.org/rest_api

***Após filtrar apenas as colunas de interesse (CHR, POS, REF e ALT),  agora podemos enviar via REST-API as variantes somaticas da amostra WP048.***

***headers -> Contém a sua credencial de acesso à API (email + token).***

***payload -> Parâmetros enviados ao CGI: **cancer_type:** tipo tumoral, ex.: HEMATO; **title:** nome do job; **reference:** versão do genoma (hg38)***

***files={'mutations': ...} -> Envia o arquivo com variantes para o CGI analisar.***

***r.json() -> Exibe a resposta da API, normalmente contendo um job_id, que será necessário para consultar o progresso.***


**NOTA:** Altere a variável {SEU-TOKEN} para o Token do CGI criado na sua conta.

```Python
import requests
headers = {'Authorization': 'eder.fersou@gmail.com {SEU_TOKEN}'}
payload = {'cancer_type': 'HEMATO', 'title': 'Somatic MF WP048', 'reference': 'hg38'}
r = requests.post('https://www.cancergenomeinterpreter.org/api/v1',
                headers=headers,
                files={
                        'mutations': open('/content/df_WP048-cgi.txt', 'rb')
                        },
                data=payload)
r.json()
```

**output Job ID:**
```
21824131b57f9e93f90d
```

**4. Verificar o status do Job (Error, Runing, Done).**

***Consulta o status do processamento: “running”, “queued”, “done”, “error”.***

```Python
import requests
job_id ="21824131b57f9e93f90d"

headers = {'Authorization': 'eder.fersou@gmail.com {SEU_TOKEN}'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers)
r.json()
```
**output:**
```
{'status': 'Done',
 'metadata': {'id': '21824131b57f9e93f90d',
  'user': 'eder.fersou@gmail.com',
  'title': 'Somatic MF WP048',
  'cancertype': 'HEMATO',
  'reference': 'hg38',
  'dataset': 'input.tsv',
  'date': '2025-12-06 14:05:18'}}
```

**5. Ver logs do JobID.**

***Retorna informações internas sobre o processamento: erros, warnings, etapas concluídas.***


```Python
import requests
job_id ="21824131b57f9e93f90d"

headers = {'Authorization': 'eder.fersou@gmail.com {SEU_TOKEN}'}
payload={'action':'logs'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
r.json()
```

**outpu:**

```
{'status': 'Done',
 'logs': ['# cgi analyze input.tsv -c HEMATO -g hg38',
  '2025-12-06 15:05:22,307 INFO     Parsing input01.tsv\n',
  '2025-12-06 15:05:26,235 INFO     Running VEP\n',
  '2025-12-06 15:05:27,147 INFO     Check cancer genes and consensus roles\n',
  '2025-12-06 15:05:27,234 INFO     Annotate BoostDM mutations\n',
  '2025-12-06 15:05:27,272 INFO     Annotate OncodriveMUT mutations\n',
  '2025-12-06 15:05:29,822 INFO     Annotate validated oncogenic mutations\n',
  '2025-12-06 15:05:29,982 INFO     Check oncogenic classification\n',
  '2025-12-06 15:05:30,048 INFO     Matching biomarkers\n',
  '2025-12-06 15:05:30,145 INFO     Prescription finished\n',
  '2025-12-06 15:05:30,160 INFO     Aggregate metrics\n',
  '2025-12-06 15:05:33,270 INFO     Compress output files\n',
  '2025-12-06 15:05:33,302 INFO     Analysis done\n']}
```

**6. Download dos resultados**

```
Total de 4 arquivos de resultados:
Definir cada um deles com base na documentação do CGI (TODOS):

alterations.tsv:
biomarker.tsv:
input01.tsv:
summary.txt:
```

***Cria a pasta onde o resultado será armazenado.***

```bash
%%bash
#criar o diretorio com o ID da amostra dentro de results
mkdir -p results/WP048
```

***Download via API***
***solicita ao CGI o pacote de resultados (ZIP) e salva o arquivo ZIP na pasta criada***

```Python
import requests
job_id ="21824131b57f9e93f90d"

headers = {'Authorization': 'eder.fersou@gmail.com {SEU_TOKEN}'}
payload={'action':'download'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
with open('/content/results/WP048/W048-cgi.zip', 'wb') as fd:
    fd.write(r._content)
```

**7. Descompactar resultados**

***Descompacta todo o conteúdo dentro da pasta results/WP048/.***

```bash
unzip -o /content/results/WP048/W048-cgi.zip -d /content/results/WP048/
```

**outpu:**
```
Archive:  /content/results/WP048/W048-cgi.zip
  inflating: /content/results/WP048/alterations.tsv  
  inflating: /content/results/WP048/biomarkers.tsv  
  inflating: /content/results/WP048/input01.tsv  
  inflating: /content/results/WP048/summary.txt
```

**8. Visualizar arquivo alterations.tsv**

***Instalar pandas***

```bash
!pip install pandas
```
**output:**

```
Requirement already satisfied: pandas in /usr/local/lib/python3.12/dist-packages (2.2.2)
Requirement already satisfied: numpy>=1.26.0 in /usr/local/lib/python3.12/dist-packages (from pandas) (2.0.2)
Requirement already satisfied: python-dateutil>=2.8.2 in /usr/local/lib/python3.12/dist-packages (from pandas) (2.9.0.post0)
Requirement already satisfied: pytz>=2020.1 in /usr/local/lib/python3.12/dist-packages (from pandas) (2025.2)
Requirement already satisfied: tzdata>=2022.7 in /usr/local/lib/python3.12/dist-packages (from pandas) (2025.2)
Requirement already satisfied: six>=1.5 in /usr/local/lib/python3.12/dist-packages (from python-dateutil>=2.8.2->pandas) (1.17.0)
```

***Ler a tabela***

***Abre a tabela principal produzida pelo CGI e mostra as alterações interpretadas (driver / passenger, fármacos associados, etc.)***

```Python
import pandas as pd
pd.read_csv('/content/results/WP048/alterations.tsv',sep='\t',index_col=False, engine= 'python')
```
**output:**


|index|Input ID|CHROMOSOME|POSITION|REF|ALT|CHR|POS|ALT\_TYPE|STRAND|CGI-Sample ID|CGI-Gene|CGI-Protein Change|CGI-Oncogenic Summary|CGI-Oncogenic Prediction|CGI-External oncogenic annotation|CGI-Mutation|CGI-Consequence|CGI-Transcript|CGI-STRAND|CGI-Type|
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
|0|input01\_1|1|114716123|C|T|chr1|114716123|snp|+|input01|NRAS|G13D|oncogenic \(predicted and annotated\)|driver \(boostDM: non-tissue-specific model\)|cgi,oncokb,clinvar:13901|chr1:114716123 C\>T|missense\_variant|ENST00000369535|+|SNV|
|1|input01\_2|9|5073770|G|T|chr9|5073770|snp|+|input01|JAK2|V617F|oncogenic \(annotated\)|passenger \(oncodriveMUT\)|cgi,oncokb,clinvar:14662|chr9:5073770 G\>T|missense\_variant|ENST00000381652|+|SNV|

Como criar uma tabela mais complexa em MarkDown
>https://www.tablesgenerator.com/markdown_tables
