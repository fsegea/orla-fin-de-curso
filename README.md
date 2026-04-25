# 🎓 Orla Fin de Curso con Software Libre

Flujo de trabajo para la generación automática de orlas de fin de curso utilizando **exclusivamente software libre** y **procesamiento 100% local**, garantizando la privacidad de las imágenes de los menores en todo momento.

## ¿Por qué este proyecto?

Las orlas escolares tradicionales implican entregar fotografías de menores a estudios fotográficos o plataformas online, con la cesión de datos que ello conlleva. Este proyecto nace de la necesidad de ofrecer una alternativa que permita a centros educativos y familias producir orlas de calidad profesional sin que las imágenes de los alumnos salgan nunca del entorno controlado del propio centro.

Todo el pipeline se ejecuta localmente mediante [ComfyUI](https://github.com/comfyanonymous/ComfyUI) y [GIMP](https://www.gimp.org/), sin dependencias de servicios en la nube ni APIs externas.

## Estructura del repositorio

```
orla-fin-de-curso/
├── workflows/
│   ├── fase_1.json          # Workflow ComfyUI: estilización de retratos
│   └── fase_2.json          # Workflow ComfyUI: recorte de fondo y reescalado
├── gimp/
│   └── PORTADA.xcf          # Plantilla GIMP para la maquetación de la orla
└── README.md
```

## Requisitos

- [ComfyUI](https://github.com/comfyanonymous/ComfyUI) instalado y en funcionamiento
- Nodos personalizados de ComfyUI:
  - [ComfyUI-Inspire-Pack](https://github.com/ltdrdata/ComfyUI-Inspire-Pack) — carga de imágenes por lotes desde directorio
  - [ComfyUI-LayerStyle](https://github.com/chflame163/ComfyUI_LayerStyle) — eliminación de fondo con BiRefNet y utilidades
- Modelos necesarios (se descargan automáticamente al abrir el workflow):
  - `qwen_image_edit_2509_fp8_e4m3fn.safetensors` — modelo de edición de imagen (UNET)
  - `qwen_2.5_vl_7b_fp8_scaled.safetensors` — codificador de texto (CLIP)
  - `qwen_image_vae.safetensors` — VAE
  - `Qwen-Image-Edit-2509-Lightning-4steps-V1.0-bf16.safetensors` — LoRA Lightning (opcional, para mayor velocidad)
  - `RMBG-2.0` — modelo de eliminación de fondo (BiRefNet)
- [GIMP](https://www.gimp.org/) 2.10 o superior para la maquetación final

---

## Fase 1 — Estilización de retratos

**Workflow:** `workflows/fase_1.json`

Esta primera fase toma las fotografías originales de los alumnos y les aplica un estilo de ilustración digital hiperrealista, obteniendo retratos uniformes y con aspecto profesional adecuados para una orla escolar.

### Directorio de entrada

Las fotografías originales deben colocarse en:

```
/home/comfyui/originales/
```

El nodo `LoadImageListFromDir` cargará todas las imágenes del directorio y las procesará en lote de forma secuencial.

### Funcionamiento interno

El workflow utiliza el modelo **Qwen Image Edit 2509** con arquitectura de difusión para transformar cada fotografía. El pipeline es el siguiente:

1. **Carga de modelos** — Se cargan el UNET (`qwen_image_edit_2509_fp8_e4m3fn`), el CLIP (`qwen_2.5_vl_7b_fp8_scaled`) y el VAE (`qwen_image_vae`).
2. **Carga de imágenes** — `LoadImageListFromDir` lee todas las imágenes del directorio de entrada.
3. **Escalado de entrada** — `FluxKontextImageScale` ajusta la imagen al tamaño óptimo para el modelo.
4. **Codificación del prompt** — `TextEncodeQwenImageEditPlus` codifica tanto el prompt positivo (estilo deseado) como el negativo junto con la imagen de referencia.
5. **Muestreo** — `KSampler` genera la imagen estilizada. El workflow incluye un switch para alternar entre configuración estándar y modo Lightning LoRA (4 pasos, más rápido).
6. **Decodificación y guardado** — El resultado se decodifica con el VAE y se guarda en el directorio de salida de ComfyUI.

### Prompt de estilización

El prompt está configurado para obtener retratos con estas características:

```
painterly hyperrealism, digital art portrait, smooth brush strokes,
Pixar-like rendering, soft gradients on skin,
digital illustrated portrait, semi-realistic painting style, smooth skin,
soft studio lighting, colorful flat background, pop art color blocks,
vibrant colors, detailed facial features, clean sharp edges,
professional yearbook portrait, painterly illustration,
high quality, 8k, centered face, neutral expression
```

Puedes modificar este prompt en el nodo `PrimitiveStringMultiline` ("Prompt") para adaptarlo al estilo que prefieras.

### Modo Lightning LoRA

El workflow incluye un switch **"Enable Lightning LoRA"** que permite alternar entre dos configuraciones del sampler:

| Parámetro | Configuración estándar | Con Lightning LoRA |
|-----------|----------------------|-------------------|
| Steps     | 20                   | 4                 |
| CFG       | 4.0                  | 1.0               |

Activa el LoRA cuando necesites procesar muchas imágenes y el tiempo sea un factor limitante.

### Directorio de salida

Las imágenes procesadas se guardan en la carpeta de salida por defecto de ComfyUI (`ComfyUI/output/`). Estas serán la entrada de la Fase 2.

---

## Fase 2 — Eliminación de fondo, centrado y reescalado

**Workflow:** `workflows/fase_2.json`

Esta segunda fase toma las imágenes estilizadas de la Fase 1, elimina el fondo automáticamente, centra al sujeto y reescala la imagen a **496 × 496 píxeles**, el formato exacto esperado por la plantilla GIMP de la orla.

### Directorio de entrada

Copia las imágenes de salida de la Fase 1 en:

```
/home/comfyui/fase_1/
```

El nodo `LoadImagesFromDir` las cargará automáticamente.

### Funcionamiento interno

1. **Carga del modelo de segmentación** — `LoadBiRefNetModelV2` carga el modelo **RMBG-2.0**, especializado en la segmentación precisa de personas.
2. **Carga de imágenes** — `LoadImagesFromDir` lee todas las imágenes del directorio de entrada.
3. **Generación de máscara** — `BiRefNetUltraV2` analiza cada imagen y genera una máscara de alta precisión que delimita la silueta del sujeto. Utiliza el método `VITMatte` para refinar los bordes.
4. **Limpieza de artefactos** — `LaMa` (nodo desactivado por defecto) puede emplearse opcionalmente para inpainting si quedan artefactos residuales del fondo.
5. **Reescalado y centrado** — `ResizeImageMaskNode` reencuadra la imagen centrando al sujeto y la escala a **496 × 496 px** usando el algoritmo Lanczos para máxima calidad.
6. **Guardado** — `SaveImage` almacena el resultado final listo para importar en GIMP.

### Parámetros de segmentación

El nodo `BiRefNetUltraV2` está configurado con los siguientes valores, que ofrecen un buen equilibrio entre precisión y velocidad:

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| Method    | VITMatte | Refinado de bordes por transformers |
| Kernel radius | 4 | Radio del kernel de suavizado |
| Expand mask   | 2 | Expansión de la máscara en píxeles |
| Black point   | 0.01 | Umbral de negro de la máscara |
| White point   | 0.99 | Umbral de blanco de la máscara |

### Salida

Imágenes PNG de **496 × 496 px** con fondo transparente, guardadas en la carpeta de salida de ComfyUI y listas para arrastrar a la plantilla GIMP.

---

## Fase 3 — Maquetación en GIMP

**Plantilla:** `gimp/PORTADA.xcf`

Abre el archivo `PORTADA.xcf` con GIMP. La plantilla está preparada para recibir directamente las imágenes de 496 × 496 px generadas en la Fase 2. Cada celda de la orla está diseñada con ese tamaño exacto, por lo que basta con arrastrar cada foto a su posición correspondiente.

---

## Flujo completo resumido

```
📁 /originales/          →  [Fase 1: Estilización]  →  📁 ComfyUI/output/
📁 /fase_1/              →  [Fase 2: Recorte+Scale]  →  📁 ComfyUI/output/ (496×496 PNG)
📁 PNGs 496×496          →  [GIMP PORTADA.xcf]       →  🎓 Orla final
```

---

## Licencia

Este proyecto se distribuye bajo licencia **GPL-3.0**, en coherencia con el software libre que utiliza. Eres libre de usar, modificar y distribuir los workflows y la plantilla respetando los términos de la licencia.

---

## Contribuciones

Las mejoras son bienvenidas. Si adaptas los workflows a otros estilos de orla, mejoras la plantilla GIMP o añades soporte para nuevos modelos, abre un Pull Request o comparte tu experiencia en Issues.
