# MLflow Deploy: Pipeline CI/CD para Wine Quality Regression

![Python](https://img.shields.io/badge/Python-3.10-blue?logo=python&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-2.10+-0194E2?logo=mlflow&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.3+-F7931E?logo=scikitlearn&logoColor=white)
![Status](https://img.shields.io/badge/status-active-brightgreen)

> Proyecto académico que automatiza por completo el ciclo de vida de un modelo de Machine Learning: desde el entrenamiento y registro con MLflow, hasta la validación contra umbrales de calidad y la publicación de artefactos auditables, todo orquestado con GitHub Actions.

---

## Resumen ejecutivo

Cada vez que se realiza un push a la rama `main`, GitHub Actions despierta y ejecuta un pipeline reproducible que:

1. Entrena un modelo de regresión sobre el dataset **Wine Quality (Red Wine)** del UCI Repository.
2. Registra el experimento en **MLflow**, incluyendo parámetros, métricas, signature e input_example.
3. Valida el modelo contra umbrales predefinidos usando datos externos.
4. Publica los artefactos en GitHub para revisión y auditoría.

Si alguna etapa falla, el pipeline se detiene y el commit queda marcado en rojo. Si todo pasa, el modelo queda disponible para descarga directa desde la pestaña Actions.

---

## ¿Por qué Wine Quality?

La elección del dataset no fue arbitraria. Estos fueron los criterios:

**Es externo.** No proviene de `sklearn.datasets`, sino del UCI Machine Learning Repository, lo que cumple el requisito explícito del taller.

**Es referencia oficial de MLflow.** El equipo de MLflow lo utiliza en su tutorial de quickstart, lo que garantiza precedente y compatibilidad probada.

**Es interpretable.** Sus 11 features físico-químicas (acidez, alcohol, pH, sulfatos, etc.) tienen significado real, lo que permite explicaciones del modelo más allá de números abstractos.

**Es del tamaño justo.** 1.599 muestras y ~85 KB: lo suficientemente pequeño para incluirlo en el repositorio sin afectar el clon, y lo suficientemente grande para entrenar un modelo no trivial.

**Es un problema de regresión real.** Predecir la calidad de un vino (escala 0-10) basándose en mediciones físicas, replicando un caso de uso industrial.

```
Dataset:        Wine Quality - Red Wine
Fuente:         UCI Machine Learning Repository
URL:            https://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/
Muestras:       1.599
Features:       11 (numéricas continuas)
Target:         quality (entero, 3-8)
Tipo:           Regresión
Tamaño:         ~85 KB
```

---

## Anatomía del proyecto

```
mlflow-deploy/
│
├── data/                            Dataset externo
│   └── winequality-red.csv          
│
├── train.py                         Entrenamiento + registro en MLflow
├── validate.py                      Validación contra umbrales
├── Makefile                         Automatización de comandos
├── requirements.txt                 Dependencias del proyecto
│
├── .github/
│   └── workflows/
│       └── mlflow-ci.yml            Workflow de GitHub Actions
│
├── mlruns/                          Tracking local de MLflow (generado)
├── model.pkl                        Modelo serializado (generado)
├── latest_run_id.txt                ID del último run (generado)
│
├── .gitignore
└── README.md
```

---

## El flujo, paso a paso

```
   Developer
       │
       │  git push origin main
       ▼
┌──────────────────────────────────────────────────────────┐
│              GitHub Actions (ubuntu-latest)              │
│                                                          │
│   ┌──────────┐   ┌──────────┐   ┌────────┐  ┌─────────┐  │
│   │ checkout │──▶│  setup   │──▶│  make  │─▶│  make   │  │
│   │   code   │   │  Python  │   │ install│  │  train  │  │
│   └──────────┘   └──────────┘   └────────┘  └────┬────┘  │
│                                                  │       │
│   ┌──────────┐   ┌──────────┐   ┌────────────┐  │       │
│   │  upload  │◀──│  upload  │◀──│    make    │◀─┘       │
│   │  mlruns  │   │  model   │   │  validate  │          │
│   └─────┬────┘   └────┬─────┘   └────────────┘          │
│         │             │                                  │
└─────────┼─────────────┼──────────────────────────────────┘
          │             │
          ▼             ▼
       Artefactos descargables en GitHub
```

---

## Cómo levantarlo localmente

### Requisitos previos

- Python 3.10 o superior
- pip
- Git

### En tres pasos

**1. Clonar y entrar al proyecto**
```bash
git clone https://github.com/cqdirecly/mlflow-deploy.git
cd mlflow-deploy
```

**2. Crear entorno y dependencias**

Linux / macOS:
```bash
python -m venv venv
source venv/bin/activate
make install
```

Windows (PowerShell):
```powershell
python -m venv venv
venv\Scripts\activate
make install
```

**3. Correr el pipeline**
```bash
make pipeline
```

¡Listo! El modelo queda entrenado, registrado y validado.

### ¿Quieres ver los resultados visualmente?

```bash
make mlflow-ui
```

Abre `http://127.0.0.1:5000` en tu navegador y explora los runs, métricas y artefactos.

---

## Catálogo de comandos (Makefile)

```bash
make help        # Muestra todos los comandos disponibles
make install     # Instala las dependencias del proyecto
make train       # Entrena el modelo y lo registra en MLflow
make validate    # Valida el modelo contra umbrales de calidad
make pipeline    # Ejecuta train + validate en secuencia
make mlflow-ui   # Lanza la interfaz visual de MLflow
make clean       # Borra artefactos generados
```

Cada target está documentado dentro del Makefile con comentarios explicativos sobre lo que hace y qué archivos genera.

---

## Componentes en profundidad

### `train.py` — El entrenador

Carga el CSV de Wine Quality, divide los datos, entrena un `RandomForestRegressor` y registra todo en MLflow. Lo que lo hace especial:

- **Modular:** funciones separadas para cargar, preparar, entrenar, evaluar y registrar.
- **Trazabilidad completa:** registra parámetros, métricas, signature (gracias a `infer_signature`) e input_example (5 filas reales).
- **Manejo de errores:** captura `FileNotFoundError` y excepciones genéricas con códigos de salida apropiados.
- **Logging profesional:** usa el módulo `logging` de Python con timestamps y niveles (no `print`).

Persiste un archivo `latest_run_id.txt` con el ID del run para que `validate.py` pueda encontrarlo después.

### `validate.py` — El guardián de calidad

Carga el modelo registrado en MLflow (no un `.pkl` con joblib, sino el modelo real desde el run con `mlflow.sklearn.load_model`) y lo evalúa con datos externos. Características clave:

- **Carga desde el registro de MLflow** usando `runs:/{run_id}/model`.
- **Datos externos:** mismo dataset pero con `random_state` distinto al de entrenamiento, simulando muestras nuevas.
- **Validación dual:** verifica tanto MSE máximo como R² mínimo.
- **Integración CI/CD:** retorna `exit code 0` si pasa, `1` si falla, lo que permite a GitHub Actions tomar decisiones automáticas.

### `mlflow-ci.yml` — El director de orquesta

Workflow que se dispara con push a `main`, pull requests, o ejecución manual. Pasos:

| # | Etapa | Comando | Función |
|---|-------|---------|---------|
| 1 | Checkout | `actions/checkout@v4` | Descarga el código en el runner |
| 2 | Python | `actions/setup-python@v5` | Instala Python 3.10 con caché |
| 3 | Dependencies | `make install` | Instala librerías |
| 4 | Train | `make train` | Entrena y registra |
| 5 | Validate | `make validate` | Valida contra umbrales |
| 6 | Artifacts | `actions/upload-artifact@v4` | Publica modelo y mlruns |

---

## Métricas y validación

### Las cuatro métricas que registramos

```
MSE   →  Mean Squared Error
RMSE  →  Root Mean Squared Error  
MAE   →  Mean Absolute Error
R²    →  Coeficiente de determinación
```

Las cuatro se calculan tanto en entrenamiento (sobre el test set) como en validación (sobre datos con random_state distinto).

### Los criterios de aprobación

```
✓ MSE  ≤  1.0    (error cuadrático medio acotado)
✓ R²   ≥  0.3    (modelo explica al menos 30% de la varianza)
```

Si **ambas** condiciones se cumplen, el modelo pasa. Si alguna falla, el pipeline se detiene y el commit queda marcado en rojo en GitHub.

---

## Trazabilidad: ¿dónde queda registrado todo?

### En MLflow (carpeta `mlruns/`)

Cada run incluye:

```
└── run_id/
    ├── metrics/         # mse, rmse, mae, r2
    ├── params/          # hiperparámetros y metadatos
    ├── tags/            # etiquetas autogeneradas
    └── artifacts/
        └── model/
            ├── MLmodel            # incluye signature
            ├── input_example.json # 5 filas reales de entrada
            ├── model.pkl
            ├── conda.yaml
            ├── python_env.yaml
            └── requirements.txt
```

### En GitHub Actions

Cada ejecución del workflow publica dos artefactos descargables que persisten 30 días:

- **`modelo-validado`** → modelo entrenado + referencia al run_id
- **`mlruns-tracking`** → tracking completo de MLflow

Esto permite que cualquier persona (incluido el evaluador) descargue los artefactos, los inspeccione localmente y reproduzca los resultados.

---

## Stack tecnológico

| Tecnología | Rol |
|------------|-----|
| **Python 3.10** | Lenguaje base |
| **scikit-learn** | Modelo (RandomForestRegressor) y métricas |
| **pandas** | Manipulación del dataset |
| **MLflow** | Tracking de experimentos y registro de modelos |
| **GitHub Actions** | Orquestación del pipeline CI/CD |
| **Make** | Automatización de comandos |

---

## Resultados típicos

Tras una ejecución exitosa, el modelo registrado típicamente alcanza:

```
Test set (random_state=42):
  MSE:  ~0.34
  RMSE: ~0.58
  MAE:  ~0.42
  R²:   ~0.46

Validación (random_state=99):
  MSE:  ~0.40
  RMSE: ~0.63  
  MAE:  ~0.45
  R²:   ~0.43
```

(Los valores exactos pueden variar ligeramente entre ejecuciones por la naturaleza del algoritmo.)

---

## Sobre el autor

**Christian Quimbay**  
Maestría en Ciencia de Datos  
Curso: MLOps

---

## Licencia

Proyecto académico desarrollado con fines educativos.
