# P8-testing-multimes

# Problema 8 · TS Training Strategy: ¿Cómo afecta un Testing en dos o más meses?

**Labo 1 · Analista Sr · Experimento colaborativo**
Grupo A: Genaro Virgilio, Santiago Alcorta · Grupo B: Lautaro Suarez, Carlos Veliz

---

## 1. Qué contiene este repositorio

| Carpeta / archivo | Contenido |
|---|---|
| `notebooks/P8_WF_Testing_Multimes_GrupoB.ipynb` | Workflow del grupo B (futuro 202107, ganancia local) |
| `notebooks/P8_WF_Testing_Multimes_GrupoA.ipynb` | Workflow del grupo A (futuro 202109, Kaggle) |
| `resultados/` | Salidas livianas de cada corrida (una carpeta por corrida) |
| `README.md` | Este archivo |

Las salidas pesadas (predicciones y modelos, ~200 MB por corrida) están en Google Drive:
**[Carpeta de resultados](https://drive.google.com/drive/folders/1w6JCo_qFGlS3uHrvdq7bFnU7kq0XsHW9?usp=sharing)**

---

## 2. Diseño experimental (resumen)

Se comparan tres configuraciones de *testing*, es decir, los meses de validación con que la Bayesian Optimization (BO) mide el AUC:

| Opción | Grupo B: testing | Grupo B: training BO | Grupo A: testing | Grupo A: training BO |
|---|---|---|---|---|
| **AA** (1 mes) = baseline | 202105 | 201901–202103 | 202107 | 201901–202105 |
| **AB** (3 meses) | 202103–202105 | 201901–202101 | 202105–202107 | 201901–202103 |
| **AC** (5 meses) | 202101–202105 | 201901–202011 | 202103–202107 | 201901–202101 |

- La producción es **idéntica al baseline** en las tres opciones: mismo preprocesamiento y feature engineering, final train con todos los meses con clase y 100% de los CONTINUA, semillerío de 10 semillas y cortes fijos de 8.000 a 14.000 envíos (de a 500).
- La única diferencia entre opciones son los hiperparámetros que encuentra cada BO (30 iteraciones, mismo espacio de búsqueda).
- Las opciones se comparan **únicamente por la ganancia de producción** (grupo B: ganancia local en 202107; grupo A: Kaggle sobre 202109). El AUC no se usa para comparar.
- Análisis final: test de Wilcoxon de rangos con signo, pareado por semilla, AB vs. AA y AC vs. AA.

Cada notebook corre las tres opciones **en bucle** (AA → AB → AC) dentro de una misma corrida, con la misma semilla primigenia, para que la comparación sea pareada.

---

## 3. Requisitos

- VM de Google Cloud creada desde el template `to-128ram-08vcpu` (8 vCPU, 128 GB de RAM). El pico de memoria está en la preparación de datos (~60–68 GB).
- Kernel de R para Jupyter (`ir`) con `data.table`, `lightgbm`, `mlrMBO`, `DiceKriging`, `primes`, `Rcpp` y `yaml`. El notebook instala los que falten.
- Dataset de la competencia en `~/datasets/analistasr_competencia_2026.csv.gz`.
- Bucket montado en `~/buckets/b1` (las salidas se escriben en `~/buckets/b1/exp/WF<experimento>/`).
- Grupo A: API de Kaggle configurada (`~/.kaggle/kaggle.json`) para la subida automática.

---

## 4. Cómo correrlo

Cada corrida es **una semilla primigenia**. Para replicar una corrida:

### 4.1 Copiar el notebook y cambiar dos líneas

En la celda de **Parámetros** (al principio del notebook) se cambian solo estas dos líneas:

```r
PARAM$semilla_primigenia <- 186653   # semilla de la corrida (ver tabla de la sección 5)
PARAM$experimento <- 10100           # número de experimento (carpeta WF<experimento>)
```

Desde la terminal de la VM:

```bash
cd ~/buckets/b1/labo2026v/ExpColaborativos     # carpeta donde está el notebook
cp P8_WF_Testing_Multimes_GrupoB.ipynb P8_WF_Testing_Multimes_GrupoB_10100.ipynb
# editar semilla y experimento (o abrir el notebook y cambiarlos a mano)
sed -i 's/semilla_primigenia <- [0-9]*/semilla_primigenia <- 186653/; s/experimento <- [0-9]*/experimento <- 10100/' P8_WF_Testing_Multimes_GrupoB_10100.ipynb
# verificar: tiene que mostrar la semilla, el experimento y seq(8000, 14000
grep -o 'semilla_primigenia <- [0-9]*\|experimento <- [0-9]*\|seq(8000, [0-9]*' P8_WF_Testing_Multimes_GrupoB_10100.ipynb
```

> Si ya existe `~/buckets/b1/exp/WF<experimento>`, el notebook **reutiliza** lo que encuentre (BO terminadas, modelos grabados). Para empezar de cero, usar otro número de experimento o renombrar esa carpeta.

### 4.2 Lanzar la corrida en segundo plano

Desde una terminal SSH de la VM (no desde la interfaz de Jupyter):

```bash
setsid nohup jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=-1 --ExecutePreprocessor.kernel_name=ir \
  P8_WF_Testing_Multimes_GrupoB_10100.ipynb > log_10100.txt 2>&1 < /dev/null &
```

- `setsid nohup ... &`: la corrida sigue aunque se cierre la terminal.
- `--inplace`: el notebook queda guardado con todas las salidas al terminar.
- Mientras corre, no abrir el notebook en Jupyter.

### 4.3 Seguimiento

```bash
EXP=10100
W=~/buckets/b1/exp/WF$EXP
ls $W/BO_log_*.txt                                                   # BO terminadas
for d in AA AB AC; do echo "$d: $(ls $W/modelos/$d 2>/dev/null | wc -l)/10"; done   # modelos finales
top -b -n 1 | head -8                                                # proceso R activo
tail -2 log_$EXP.txt                                                 # al final: "Writing ... bytes to ..."
```

Terminó cuando existe `resultado_opciones.txt` y el log cierra con `[NbConvertApp] Writing ... bytes to ...`.

**Tiempo por corrida:** ~11–17 h con 8 vCPU (preparación ~35 min, 3 BO ~4–6 h, 30 modelos finales ~6–12 h según los hiperparámetros elegidos).

**Si la VM se cae:** relanzar el mismo comando. La preparación de datos se recalcula, pero las BO terminadas (`BO_log_<opción>.txt`), las BO a medias (`HT_<opción>.RDATA`) y los modelos ya grabados (`modelos/<opción>/`) se retoman.

---

## 5. Corridas realizadas

| Grupo | Experimento | Semilla primigenia | Integrante |
|---|---|---|---|
| B | 10100 | 186653 | Lautaro Suarez |
| B | 10101 | 315779 | Lautaro Suarez |
| B | 10101 | 135007 | Carlos Veliz |
| B | 10102 | 631861 | Lautaro Suarez |
| B | 10103 | 667657 | Lautaro Suarez |
| B | 10104 | 932227 | Lautaro Suarez |
| B | 10105 | 699967 | Lautaro Suarez |
| B | 10106 | 400009 | Lautaro Suarez |
| B | 10106 | 555557 | Genaro Virgilio |
| B | _completar_ | _completar_ | _completar_ |
| A | 10100 | 104759 | Santiago Alcorta |
| A | 10101 | 999931 | Santiago Alcorta |
| A | 10102 | 999953 | Santiago Alcorta |
| A | 10103 | 999959 | Santiago Alcorta |
| A | 10104 | 999979 | Santiago Alcorta |
| A | 10100 | 149911 | Genaro Virgilio |
| A | 10101 | 234571 | Genaro Virgilio |
| A | 10102 | 345679 | Genaro Virgilio |
| A | 10103 | 567899 | Genaro Virgilio |
| A | 10104 | 322633 | Genaro Virgilio |

> Corridas con el mismo número de experimento pertenecen a integrantes distintos y se ejecutaron en buckets distintos.

---

## 6. Salidas de cada corrida

En `~/buckets/b1/exp/WF<experimento>/`:

| Archivo | Contenido | En el repo |
|---|---|---|
| `resultado_opciones.txt` | Grupo B: ganancia en cada corte fijo (`gan_8000` … `gan_14000`) por opción. Grupo A: AUC de la BO (solo registro) | sí |
| `BO_log_AA/AB/AC.txt` | Todas las iteraciones de cada BO y sus hiperparámetros | sí |
| `PARAM.yml` | Parámetros, hiperparámetros finales y tiempos | sí |
| `curva_de_ganancia.pdf` | Grupo B: curvas de ganancia de las tres opciones | sí |
| `impo_AA/AB/AC.txt`, `canaritos.txt` | Importancia de variables | sí |
| `envios_kaggle.txt` | Grupo A: índice de los 39 archivos enviados (3 opciones × 13 cortes) | sí |
| `kaggle_resultados_<exp>.csv` | Grupo A: Public score de cada envío, bajado con la API de Kaggle | sí |
| `curva_AA/AB/AC.txt` | Grupo B: ganancia acumulada y suavizada cada 50 envíos (versión reducida de `ganancias_*.txt`) | sí |
| `ganancias_*.txt`, `prediccion_*.txt` | Ganancia y probabilidades por cliente | Drive |
| `modelos/<opción>/modelo_<semilla>.txt` | 10 modelos finales por opción | Drive |
| `HT_*.RDATA`, `modelo.model` | Estado interno de la BO y del FE | no |

### Bajar los scores de Kaggle (grupo A)

```bash
kaggle competitions submissions -c labo-1-virtual-2026-analista-sr -v > kaggle_todas.csv
head -1 kaggle_todas.csv > kaggle_resultados_10100.csv
grep "KA10100_" kaggle_todas.csv >> kaggle_resultados_10100.csv
```

---

## 7. Análisis

1. Para cada corrida y opción se calcula **un solo valor**: el promedio de la ganancia de producción en los 13 cortes fijos (8.000 a 14.000).
2. Por semilla se calculan las diferencias AB − AA y AC − AA.
3. Se aplica el test de Wilcoxon de rangos con signo, pareado por semilla, **por grupo** (las escalas de A y B no son comparables):

```r
# tb: una fila por semilla, columnas AA, AB, AC (ganancia promedio de los 13 cortes)
wilcox.test(tb$AB, tb$AA, paired = TRUE, alternative = "greater")
wilcox.test(tb$AC, tb$AA, paired = TRUE, alternative = "greater")
```
