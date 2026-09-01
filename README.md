# Lector de Facturas — FacturDor OCR

Servicio OCR liviano para facturas térmicas colombianas. Parte del MVP:

> **Telegram foto → n8n → `POST /parse` → LLM categoriza → Telegram respuesta**

Motor: **RapidOCR ONNX** (~600MB, CPU, `lang=es`) — optimizado para VPS 2GB.

## Estructura

```
.
├── ocr-service/
│   ├── app.py              # FastAPI + RapidOCR + parser heurístico
│   ├── requirements.txt    # rapidocr-onnxruntime 1.4.2
│   └── Dockerfile          # python:3.10-slim
├── docker-compose.yml
├── openapi.yaml            # Contrato OpenAPI 3.0
└── README.md
```

## Quick start (local)

```bash
docker compose up --build -d
curl http://localhost:8000/health
# → {"status":"ok","engine":"rapidocr-onnxruntime"}

# OCR crudo
curl -X POST "http://localhost:8000/ocr?min_confidence=0.4" \
  -F "file=@factura.jpg"

# OCR + parsing (usado por n8n)
curl -X POST "http://localhost:8000/parse?min_confidence=0.4" \
  -F "file=@factura.jpg" | jq
```

Fotos: apoyá la factura en mesa oscura, luz difusa arriba, sin flash ni dedo tapando. Con eso pasás de 60% a 90% de acierto.

## API

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/health` | Health check |
| `POST` | `/ocr` | OCR crudo: `text` + `segments` con `confidence` y `bbox` |
| `POST` | `/parse` | OCR + parsing heurístico: `ocr_text` + `parsed` (NIT, fecha, productos, totales) |

Query `min_confidence` (0-1, default 0.4) filtra `filtered_segments`.

Contrato completo: [`openapi.yaml`](./openapi.yaml) — importable en Swagger UI / Postman.

### Ejemplo `/parse`

```json
{
  "status": "success",
  "engine": "rapidocr-onnxruntime",
  "ocr_text": "Fecha:31/08/2026\n1.44Kg 15400-7012-PECHUGA BLANCA 0 22176\n...",
  "parsed": {
    "fecha": "31/08/2026",
    "comercio": { "nit": "800106714-0", "nombre": "MERCADO ZAPATOCA S.A" },
    "productos": [
      { "cantidad": "1.44Kg", "codigo": "15400-7012", "descripcion": "PECHUGA BLANCA", "iva": "0", "total": "22176", "raw": "..." }
    ],
    "totales_detectados": { "efectivo_entregado": "400,000", "cambio": "17,350" },
    "conteo_productos": 4
  },
  "llm_hint": "Usá `parsed.productos` y `ocr_text` para categorizar..."
}
```

> `parsed` es best-effort. La fuente de verdad para el LLM es `ocr_text` + `segments`.

## n8n — Workflow MVP

1. **Telegram Trigger** — On Message (foto)
2. **HTTP Request** — `POST http://ocr-service:8000/parse` — Binary `file` = foto de Telegram
3. **AI Agent (OpenAI)** — Prompt:
   ```
   Extraé JSON {fecha, productos[{cantidad, descripcion, total, categoria}], totales}
   de ocr_text. Corregí OCR (PERR0→PERRO, CALD0→CALDO) y categorizá
   (carnes, panadería, frutas, pastas, caldos...). Devolvé JSON válido.
   ```
4. **[Opcional]** Google Sheets / Postgres — guardar JSON
5. **Telegram Send** — `🧾 {{fecha}} — {{conteo}} productos — Total ${{total}}`

El servicio es `http://ocr-service:8000` dentro de la network `facturdor-net`.

## VPS (producción)

```bash
scp -r ocr-service docker-compose.yml openapi.yaml user@vps:/opt/facturdor
ssh user@vps
cd /opt/facturdor && docker compose up --build -d
```

Recursos: 2GB RAM mínimo, 4GB ideal. Imagen ~600MB. No necesita GPU.

**No expongas `8000` a internet** — dejalo en `127.0.0.1:8000` o detrás de Nginx/Caddy con HTTPS y auth. En `docker-compose.yml`:
```yaml
ports: ["127.0.0.1:8000:8000"]
```

## Troubleshooting

| Error | Causa | Fix |
|-------|-------|-----|
| `Expected UploadFile, received str` | `-F "file=path"` sin `@` | Usa `-F "file=@/path/factura.jpg"` y `curl.exe` en PowerShell |
| `PIL.Image has no attribute ANTIALIAS` | Pillow 10+ con easyocr viejo | Ya migrado a RapidOCR, no aplica |
| `productos: []` | Filas separadas en columnas | Usa `ocr_text` para el LLM; el parser agrupa por Y (12px) best-effort |
| Build lento / OOM | VPS 1GB | Subí a 2GB, RapidOCR necesita ~800MB en build |

## Roadmap

- [x] OCR RapidOCR + /parse
- [ ] Prompt n8n para categorías y presupuesto
- [ ] Backend con estadísticas y control de presupuesto
