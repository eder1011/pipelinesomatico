# pipelinesomatico
Pipeline Somático - Do VCF (anotado) até o CGI Classificação

**1. Clonar o git Lmabrasil-hg38**

```bash
! git clone https://github.com/renatopuga/lmabrasil-hg38.git
```
output: 

```
Cloning into 'lmabrasil-hg38'...
remote: Enumerating objects: 226, done.
remote: Counting objects: 100% (168/168), done.
remote: Compressing objects: 100% (108/108), done.
remote: Total 226 (delta 90), reused 114 (delta 56), pack-reused 58 (from 1)
Receiving objects: 100% (226/226), 8.63 MiB | 32.38 MiB/s, done.
Resolving deltas: 100% (106/106), done.
```

Agora vá até o github Imabrasil-hg38 na seção USando CGI via API Rest no google Colab


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

output: 

```
CHR	POS	REF	ALT
chr1	114716123	C	T
chr9	5073770	G	T
CHR	POS	REF	ALT
chr1	114716123	C	T
chr9	5073770	G	T
```

Enviar Job para Cancer Genome Interpreter (CGI) API
>https://www.cancergenomeinterpreter.org/rest_api
>
>Após filtrar apenas as colunas de interesse (CHR, POS, REF e ALT),  agora podemos enviar via REST-API as variantes somaticas da amostra WP048.
>
>NOTA: Altere a variável {SEU-TOKEN} para o TOKEN DO cgi CRIADO POR SUA CONTA
>
>```Python
>import requests
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

output Job ID:
```
21824131b57f9e93f90d
```

Status do JOB ID (Error, Runing, done)

```Python
import requests
job_id ="21824131b57f9e93f90d"

headers = {'Authorization': 'eder.fersou@gmail.com {SEU_TOKEN}'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers)
r.json()
```
Output:
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

Log completo do JOBID

```
import requests
job_id ="21824131b57f9e93f90d"

headers = {'Authorization': 'eder.fersou@gmail.com e0122907ce53686b5d72'}
payload={'action':'logs'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
r.json()
```

Outpu:

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

Download dos Resultados

```
Total de 4 arquivos de resultados:

> Definir cada um deles com base na documentação do CGI (TODOS):

> alterations.tsv
> biomarker.tsv
> input01.tsv
> summary.txt
```

```
%%bash
#criar o diretorio com o ID da amostra dentro de results
mkdir -p results/WP048
```
```
import requests
job_id ="21824131b57f9e93f90d"

headers = {'Authorization': 'eder.fersou@gmail.com e0122907ce53686b5d72'}
payload={'action':'download'}
r = requests.get('https://www.cancergenomeinterpreter.org/api/v1/%s' % job_id, headers=headers, params=payload)
with open('/content/results/WP048/W048-cgi.zip', 'wb') as fd:
    fd.write(r._content)
```

**Descompactar o zip com os resultados**

```
%%bash
unzip /content/results/WP048/W048-cgi.zip -d /content/results/WP048/
```

outpu:
```
Archive:  /content/results/WP048/W048-cgi.zip
  inflating: /content/results/WP048/alterations.tsv  
  inflating: /content/results/WP048/biomarkers.tsv  
  inflating: /content/results/WP048/input01.tsv  
  inflating: /content/results/WP048/summary.txt
```
```Python
import pandas as pd
pd.read_csv('/content/results/WP048/alterations.tsv',sep='\t',index_col=False, engine= 'python')
```

Como criar uma tabela mais complexa em MarkDown
>https://www.tablesgenerator.com/markdown_tables
