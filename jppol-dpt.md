# Migrering
I hovedtræk skal følgende gøres:
1. Flyt kode og dag filer til hhv. `src/` og `.airflow/`
2. `uv`-opsætning
3. Tilføj .pre-commit-config.yaml
4. Opdater .Dockerfile og CICD-filer
5. Opdater dags
6. Opdater readme

OBS!
> FØR migrering:
> - Arkiver det gamle repo inden du kloner til jppol-dpt - omdøb det nye repo til noget med prefix `etl.`. Lav klon med historik sådan:
>   ```bash
>   cd ~
>   git clone https://github.com/ebanalyse/<GAMMEL_REPO> abc
>   cd abc
>   
>   for remote in `git branch -r | grep -v master `; \
>   do git checkout --track $remote ; done
>   
>   git remote add neworigin https://github.com/jppol-dpt/<NYT_REPO>
>   git push --all neworigin
>   ```
> - Lav én branch til ændringerne, og lad en PR håndterer merge
> - Inden PR merges med main bør eksisterende DAGS sættes på pause, for at undgå konflikt.

> Test:
> - Kontroller data er som forventet (row count/ stikprøve)
> - Kontroller at cloudwatch log ikke viser fejl

> EFTER migrering:
> - Slet gamle DAG-filer manuelt


## 1. Flyt kode
- Læg kode i `src/`. Hvis ikke du bygger pakker, så undgå at lægge koden i "pakke"-foldere.
- Flyt dags til `.airflow/`. Hvis ikke stien følger mønstret `airflow/dag.py` skal CD steppet som kopierer DAGS gives input til anden sti.

## 2. UV opsætning
- Opdater `pyproject.toml`, så det er kompatibelt med "uv". Se øvrige "jppol-dpt"-repos.
- Kør `uv sync`. Du får fejl 401 når uv ikke kan kommunikere med AWS Codeartifact. Dette er nødvendigt så længe uv skal løse afhængigheder gennem `DOS_COMMON`. I dette tilfælde kræver `uv` to specifikke env vars i din shell. Kør `source script.sh` hvor `script.sh` indeholder dette:
```shell
export UV_INDEX_DOS_COMMON_USERNAME=aws
export UV_INDEX_DOS_COMMON_PASSWORD="$(
    aws codeartifact get-authorization-token \
    --domain jppol-dos \
    --domain-owner 570306902459 \
    --profile dfp-dev-role \
    --region eu-west-1 \
    --query 'authorizationToken' \
    --output text
)
```
- Kør `uv sync` igen.

## 3. Pre-commit
For at sikre ensartethed i kodestruktur og -kvalitet, bruger vi "pre-commit":
- Hvis pre-commit endnu ikke er tilføjet til projektet, kør da `uv add pre-commit --dev`
- Kør `uv run pre-commit install` for at lade git køre pre-commit som et commit-hook 

Nu vil pre-commit køres inden hvert commit!

- Fordi pre-commit kun er automatisk for de filer der committes ændringer til, bør kommandoen `uv run pre-commit run --all-files` køres som led af migrering, da det så får alle filer up-to-date. 

## 4. .Dockerfile og .github/
- Såfremt projektet er __klassisk__ pipeline, kan samme `.Dockerfile` og CICD filer kopieres fra allerede-migrerede repos. Er det ikke tilfældet, kræver det individuel tilpasning.  
> Repos med prefix `etl.` kræver PR, og at PR automatisk kører et pre-commit check af seneste commit.

## 5. Dags
- Flyt dags til `.airflow/`. Hvis ikke dag filer ligger der, bør CD-steppet der deployer DAGS have input med korrekt sti.
- I fht. navngivning af Airflow og Batch referencer bør opsætningen fra allerede-migrerede repos følges. Dette gøres bl.a. for ikke at skulle manuelt angive navn på batch_job_definition i det tilsvarende CD-workflow.
- Note: Default CPU og RAM for batch jobs er 1 kerne/GB.

## 6. Readme
- Lav et overhaul af dokumentationen. Som minimum bør du give lidt kontekst til projektet, samt relevante CLI commands. I fht. CLI må udstilling af samtlige variationer af commands gerne undgås. Brug i steder for denne [guide](https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/V1_chap12.html#tag_12_01) til at vise variation.