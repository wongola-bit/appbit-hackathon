# App BiT — Base de Dados v2
## CDRView · Região Metropolitana de Florianópolis · jun/2026

---

## Novidades na v2 (08/06/2026)

Dois arquivos foram adicionados sem alterar os anteriores:

- **tensor_sequencias.csv** — sequência ordenada de antenas visitadas por
  cada assinante em cada dia, com `arrival_time` e distância percorrida.
  Permite análise de trajetos individuais e identificação de vias urbanas.

- **tensor_fluxo_vias.csv** — pares de antenas consecutivas agregados com
  volume de usuários e percentual de fluxo. Permite visualização de
  corredores e identificação de gargalos em vias.

Os arquivos da v1 (`tensor_mobilidade`, `trajetos_comuns`, etc.)
**não foram alterados**.

---

## Conteúdo desta pasta

| Arquivo | Tamanho | O que é |
|---------|---------|---------|
| `CDRView_AppBiT_TechnicalReference.docx` | ~30 KB | **Leia primeiro.** Schema completo, glossário e queries SQL. |
| `tensor_mobilidade.csv` | ~2,7 GB | Base principal — 16,8M eventos de 200K assinantes em 15 dias. |
| `tensor_sequencias.csv` | ~915 MB | Sequência de antenas por assinante/dia com arrival_time. **NOVO** |
| `bases_hackathon_bit.zip` | ~3 MB | Todos os demais CSVs (ver lista abaixo). |

### Conteúdo do bases_hackathon_bit.zip

| Arquivo | Linhas | O que é |
|---------|--------|---------|
| `antenas_flp.csv` | 132 | ERBs reais da Claro na RM (fonte: Anatel) |
| `assinantes.csv` | 200.000 | Perfil demográfico de cada assinante |
| `tensor_concentracao.csv` | 7.920 | Concentração por antena, dia e período |
| `tensor_od.csv` | ~500 | Pares Origem-Destino entre clusters |
| `tensor_fluxo_vias.csv` | ~15.000 | Pares de antenas consecutivas com fluxo. **NOVO** |
| `tensor_tempo_deslocamento.csv` | ~460 | Distâncias inter-cluster |
| `trajetos_comuns.csv` | ~500 | Pares OD k-anonimizados (K=3) |
| `sumario_kanon.csv` | 6 | Relatório de conformidade de privacidade |

---

## Quando usar cada arquivo

| Análise | Arquivo principal |
|---------|------------------|
| Mapa de calor de concentração | `tensor_concentracao.csv` |
| Fluxo entre bairros / zonas | `tensor_od.csv` ou `trajetos_comuns.csv` |
| Carga em vias e corredores | `tensor_fluxo_vias.csv` |
| Trajeto completo de um assinante | `tensor_sequencias.csv` |
| Segmentação demográfica | `assinantes.csv` |
| Análise de qualidade de rede | `tensor_mobilidade.csv` |
| Tempo médio entre zonas | `tensor_tempo_deslocamento.csv` |

---

## Como montar o ambiente

### Python / pandas

```python
import pandas as pd

# Arquivos pequenos — carregue completo
antenas      = pd.read_csv("antenas_flp.csv", dtype={"ecgi": str})
assinantes   = pd.read_csv("assinantes.csv")
concentracao = pd.read_csv("tensor_concentracao.csv", dtype={"ecgi": str})
od           = pd.read_csv("tensor_od.csv")
fluxo_vias   = pd.read_csv("tensor_fluxo_vias.csv", dtype={"ecgi_origem": str,
                                                             "ecgi_destino": str})

# Arquivos grandes — SEMPRE em chunks
for chunk in pd.read_csv("tensor_mobilidade.csv",
                          chunksize=500_000,
                          dtype={"ecgi": str, "assinante_hash": "int32"}):
    pass  # seu processamento

for chunk in pd.read_csv("tensor_sequencias.csv",
                          chunksize=500_000,
                          dtype={"ecgi": str, "assinante_hash": "int32"},
                          parse_dates=["arrival_time"]):
    pass  # seu processamento
```

> **Atenção:** sempre leia colunas `ecgi`, `ecgi_origem`, `ecgi_destino`
> como `str`. O pandas converte para float64 por padrão e corrompe o identificador.

### Oracle SQL

```sql
-- Top corredores por volume de usuarios
SELECT ecgi_origem, cluster_origem, ecgi_destino, cluster_destino,
       n_usuarios, n_transicoes, dist_km, pct_do_cluster_origem
FROM tensor_fluxo_vias
ORDER BY n_usuarios DESC
FETCH FIRST 20 ROWS ONLY;

-- Sequencia de um assinante num dia especifico
SELECT seq_num, ecgi, cluster, arrival_time,
       permanencia_seg, distancia_km_anterior
FROM tensor_sequencias
WHERE assinante_hash = 12345
  AND day_date = DATE '2026-03-05'
ORDER BY seq_num;
```

---

## Schema resumido dos arquivos novos

### tensor_sequencias.csv

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `assinante_hash` | INT | Identificador do assinante |
| `day_date` | DATE | Data (YYYY-MM-DD) |
| `seq_num` | INT | Posição na sequência do dia (1, 2, 3...) |
| `ecgi` | STRING | Antena visitada — sempre tratar como string |
| `cluster` | STRING | Zona geográfica da antena |
| `municipio` | STRING | Município da antena |
| `lat` | FLOAT | Latitude da antena |
| `lon` | FLOAT | Longitude da antena |
| `arrival_time` | DATETIME | Timestamp da 1ª sessão na antena (ISO 8601) |
| `permanencia_seg` | INT | Tempo estimado de permanência em segundos |
| `periodo_sessao` | STRING | MADRUGADA / MANHA / TARDE / NOITE |
| `distancia_km_anterior` | FLOAT | Distância Haversine da antena anterior (0 na 1ª) |
| `n_sessoes` | INT | Volume de sessões de dados nessa antena no dia |

> `arrival_time` é sintético — gerado dentro da janela do `periodo_sessao`
> com distribuição que favorece o início do período.
> Em produção, o CDRView substitui pelo timestamp real do primeiro CDR da célula.

### tensor_fluxo_vias.csv

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `ecgi_origem` | STRING | Antena de origem do deslocamento |
| `lat_origem` | FLOAT | Latitude da antena de origem |
| `lon_origem` | FLOAT | Longitude da antena de origem |
| `cluster_origem` | STRING | Zona geográfica de origem |
| `municipio_origem` | STRING | Município de origem |
| `ecgi_destino` | STRING | Antena de destino do deslocamento |
| `lat_destino` | FLOAT | Latitude da antena de destino |
| `lon_destino` | FLOAT | Longitude da antena de destino |
| `cluster_destino` | STRING | Zona geográfica de destino |
| `municipio_destino` | STRING | Município de destino |
| `n_usuarios` | INT | Usuários distintos que fizeram este par |
| `n_transicoes` | INT | Total de transições observadas |
| `dist_km` | FLOAT | Distância Haversine entre as antenas em km |
| `periodo_predominante` | STRING | Período mais frequente para este par |
| `pct_do_cluster_origem` | FLOAT | % dos usuários da antena de origem neste destino |

---

## Períodos do dia

| Código | Horário | Perfil |
|--------|---------|--------|
| `MADRUGADA` | 00h–06h | Baixo uso (8% dos eventos) |
| `MANHA` | 06h–12h | Deslocamento para trabalho (28%) |
| `TARDE` | 12h–18h | Pico de uso (35%) |
| `NOITE` | 18h–00h | Lazer e streaming (29%) |

---

## Privacidade e K-anonimato

Base gerada com **K=3** (dados sintéticos, hackathon).
Em produção com dados reais: **K=5 obrigatório** (LGPD Art. 12).
Arquivo `sumario_kanon.csv` documenta a conformidade.

---

*Vísent · OSX Telecomunicações S/A · Hackathon App BiT · jun/2026*
*Parceria: Wongola · Angola Cables · Oracle · PMI-SP*
