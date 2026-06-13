# Malaria Triage AI — API Contract

**Base URL**: `http://localhost:8000`

---

## `GET /health`

Health check.

### Response `200`
```json
{
  "status": "ok"
}
```

---

## `POST /api/v1/predict-batch`

Upload de 1 a N imagens de lâminas de sangue para triagem.

### Request

`multipart/form-data`

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `files` | `file[]` | Lista de imagens JPEG ou PNG |

### Limites (configuráveis via env)

| Parâmetro | Default | Descrição |
|-----------|---------|-----------|
| `max_batch_images` | 20 | Máximo de imagens por requisição |
| `max_upload_mb` | 10 | Tamanho máximo por arquivo (MB) |
| `accepted_mime_types` | `image/jpeg, image/png` | Tipos MIME aceitos |

### Pipeline de processamento por imagem

```
Upload → validar MIME → validar tamanho → decodificar (OpenCV) →
validar qualidade (blur, contraste, brilho, dimensões mínimas) →
realçar imagem (ROI circular, white balance, CLAHE, denoise, sharpen) →
pré-processar tensor (224×224, normalizar) → inferência ONNX →
classificar score → agregar scores (média)
```

### Response `200`

```json
{
  "label": "suspected_positive",
  "confidence": 0.8732,
  "images_processed": 3,
  "images_rejected": 0,
  "model_version": "resnet18_v1"
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `label` | `string` | `"suspected_positive"` ou `"likely_negative"` |
| `confidence` | `float` | Score entre 0.0 e 1.0 |
| `images_processed` | `int` | Imagens que passaram na validação |
| `images_rejected` | `int` | Imagens rejeitadas (qualidade/decode) |
| `model_version` | `string` | Versão do modelo utilizado |

#### Regra de classificação

| Score | Label |
|-------|-------|
| >= 0.80 | `suspected_positive` |
| < 0.80 | `likely_negative` |

> O label `uncertain` existe no enum, mas **não é retornado** — foi removido da lógica de classificação.

### Errors

| Status | `error` | `message` | Causa |
|--------|---------|-----------|-------|
| 400 | `no_images_provided` | `At least one image file is required.` | Nenhum arquivo enviado |
| 400 | `too_many_images` | `The request exceeds the maximum batch size of N.` | > 20 imagens |
| 413 | `file_too_large` | `One or more uploaded files exceed the maximum allowed size.` | Arquivo > 10 MB |
| 415 | `unsupported_media_type` | `Only JPG and PNG images are supported.` | MIME não aceito |
| 422 | `image_decode_failed` | `OpenCV não conseguiu decodificar...` | Todas as imagens falharam no decode |
| 422 | `image_quality_insufficient` | `Nenhuma imagem passou nos critérios mínimos...` | Todas rejeitadas por qualidade |
| 500 | `model_not_loaded` | `...` | Modelo ONNX não carregado |
| 500 | `inference_failed` | `The model could not process the request.` | Erro na inferência |

Payload de erro:
```json
{
  "error": "too_many_images",
  "message": "The request exceeds the maximum batch size of 20."
}
```

---

## `GET /api/v1/stats/dashboard`

> ⚠️ **Stub** — retorna dados fixos. Sem banco de dados implementado.

### Response `200`

```json
{
  "status": "ok",
  "summary": {
    "total_occurrences": 42,
    "suspected_positive": 17,
    "uncertain": 9,
    "likely_negative": 16
  },
  "time_series": [
    {
      "day": "2026-05-14",
      "suspected_positive": 2,
      "uncertain": 1,
      "likely_negative": 3
    },
    {
      "day": "2026-05-15",
      "suspected_positive": 4,
      "uncertain": 2,
      "likely_negative": 3
    },
    {
      "day": "2026-05-16",
      "suspected_positive": 1,
      "uncertain": 3,
      "likely_negative": 6
    }
  ],
  "model_version": "resnet18_v1"
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `status` | `string` | `"ok"` |
| `summary` | `object` | Totais agregados |
| `summary.total_occurrences` | `int` | Total de triagens |
| `summary.suspected_positive` | `int` | Total suspeitos positivos |
| `summary.uncertain` | `int` | Total incertos |
| `summary.likely_negative` | `int` | Total provavelmente negativos |
| `time_series` | `array` | Série histórica por dia |
| `time_series[].day` | `string` | Data (`YYYY-MM-DD`) |
| `time_series[].suspected_positive` | `int` | Suspeitos no dia |
| `time_series[].uncertain` | `int` | Incertos no dia |
| `time_series[].likely_negative` | `int` | Negativos no dia |
| `model_version` | `string` | Versão do modelo |

---

## `GET /api/v1/occurrences`

> ⚠️ **Stub** — retorna dados fixos. Sem banco de dados implementado.

### Response `200`

```json
{
  "status": "ok",
  "items": [
    {
      "id": "occ_001",
      "created_at": "2026-05-16T10:12:30Z",
      "label": "suspected_positive",
      "confidence": 0.91
    },
    {
      "id": "occ_002",
      "created_at": "2026-05-16T10:18:02Z",
      "label": "uncertain",
      "confidence": 0.54
    },
    {
      "id": "occ_003",
      "created_at": "2026-05-16T10:22:49Z",
      "label": "likely_negative",
      "confidence": 0.12
    }
  ],
  "limit": 20,
  "offset": 0,
  "total": 3
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `status` | `string` | `"ok"` |
| `items` | `array` | Lista de ocorrências |
| `items[].id` | `string` | ID único da triagem |
| `items[].created_at` | `string` | Timestamp ISO 8601 |
| `items[].label` | `string` | `suspected_positive` / `uncertain` / `likely_negative` |
| `items[].confidence` | `float` | Score entre 0.0 e 1.0 |
| `limit` | `int` | Máximo por página (hardcoded 20) |
| `offset` | `int` | Deslocamento (hardcoded 0) |
| `total` | `int` | Total de registros |

---

## Variáveis de Ambiente

| Variável | Default | Descrição |
|----------|---------|-----------|
| `APP_NAME` | `Malaria Triage AI` | Nome da aplicação |
| `API_BASE_PATH` | `/api/v1` | Prefixo base da API |
| `MODEL_PATH` | `model/malaria_resnet18.onnx` | Caminho do modelo ONNX |
| `MODEL_VERSION` | `resnet18_v1` | Versão do modelo |
| `DEMO_MODE` | `false` | `true` = score fake (sem modelo real) |
| `INPUT_SIZE` | `224` | Tamanho do tensor de entrada |
| `POSITIVE_THRESHOLD` | `0.80` | Threshold para suspeito positivo |
| `NEGATIVE_THRESHOLD` | `0.30` | Threshold para negativo (não usado atualmente) |
| `MAX_UPLOAD_MB` | `10` | Tamanho máximo por arquivo |
| `MAX_BATCH_IMAGES` | `20` | Máximo de imagens por requisição |
| `ACCEPTED_MIME_TYPES` | `image/jpeg,image/png` | Tipos MIME permitidos |
| `MIN_IMAGE_WIDTH` | `64` | Largura mínima da imagem |
| `MIN_IMAGE_HEIGHT` | `64` | Altura mínima da imagem |
| `BLUR_THRESHOLD` | `100` | Threshold de blur (Laplacian) |
| `MIN_CONTRAST` | `15` | Contraste mínimo (std dev) |
| `MIN_BRIGHTNESS` | `30` | Brilho mínimo (mean) |
| `MAX_BRIGHTNESS` | `225` | Brilho máximo (mean) |

---

## Considerações Técnicas

- **CORS**: Liberado para qualquer origem (`allow_origins=["*"]`) — desenvolvimento.
- **Modelo ONNX**: ResNet18, entrada `[1, 3, 224, 224]` float32, saída `[1, 2]` float32 (logits para parasitada vs não infectada).
- **Suporte a 3 formatos de saída**: sigmoid único, probabilidades 2-class, logits 2-class (converte com softmax).
- **Demo mode**: `DEMO_MODE=true` retorna score baseado na média de intensidade dos pixels — útil para testar o frontend sem o modelo.
- **Sem autenticação**: API aberta, sem tokens ou chaves.
- **Sem banco de dados**: `stats/dashboard` e `/occurrences` são stubs com dados hardcoded.