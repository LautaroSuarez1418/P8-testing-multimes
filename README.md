# Problema 8 · TS Training Strategy: ¿Cómo afecta un Testing en dos o más meses?

**Labo 1 · Analista Sr · Experimento colaborativo**
Grupo A: Genaro Virgilio, Santiago Alcorta · Grupo B: Lautaro Suarez, Carlos Veliz

---

## 1. Qué contiene este repositorio

| Carpeta / archivo | Contenido |
|---|---|
| `notebooks/P8_WF_Testing_Multimes_GrupoB.ipynb` | Workflow del grupo B (futuro 202107, ganancia local): AA, A2, AB y AC en bucle |
| `notebooks/P8_WF_Testing_Multimes_GrupoA.ipynb` | Workflow del grupo A (futuro 202109, Kaggle): AA, A2, AB y AC en bucle |
| `resultados/` | Salidas livianas de cada corrida (una carpeta por corrida, 40 en total) |
| `README.md` | Este archivo |

Las salidas pesadas (predicciones y modelos, ~200 MB por corrida) están en Google Drive:
**[Carpeta de resultados](https://drive.google.com/drive/folders/1w6JCo_qFGlS3uHrvdq7bFnU7kq0XsHW9?usp=sharing)**

---

## 2. Diseño experimental (resumen)

Se comparan cuatro configuraciones de *testing*, es decir, los meses de validación con que la Bayesian Optimization (BO) mide el AUC:

| Opción | Grupo B: testing | Grupo B: training BO | Grupo A: testing | Grupo A: training BO |
|---|---|---|---|---|
| **AA** (1 mes) = baseline | 202105 | 201901–202103 | 202107 | 201901–202105 |
| **A2** (2 meses) | 202104–202105 | 201901–202102 | 202106–202107 | 201901–202104 |
| **AB** (3 meses) | 202103–202105 | 201901–202101 | 202105–202107 | 201901–202103 |
| **AC** (5 meses) | 202101–202105 | 201901–202011 | 202103–202107 | 201901–202101 |

- La producción es **idéntica al baseline** en las cuatro opciones: mismo preprocesamiento y feature engineering, final train con todos los meses con clase y 100% de los CONTINUA, semillerío de 10 semillas y cortes fijos de 8.000 a 14.000 envíos (de a 500).
- La única diferencia entre opciones son los hiperparámetros que encuentra cada BO (30 iteraciones, mismo espacio de búsqueda).
- Las opciones se comparan **únicamente por la ganancia de producción** (grupo B: ganancia local en 202107; grupo A: Public de Kaggle sobre 202109). El AUC no se usa para comparar.
- Análisis final: test de Wilcoxon de rangos con signo, pareado por semilla (A2, AB y AC contra AA), y test de Friedman con las cuatro opciones.

Cada notebook corre las cuatro opciones **en bucle** (AA → A2 → AB → AC) dentro de una misma corrida, con la misma semilla primigenia, para que la comparación sea pareada.

> **A2 se agregó después.** En las corridas de este repositorio, AA, AB y AC corrieron juntas (experimentos 101xx) y A2 en una corrida aparte (experimentos 201xx) con la **misma semilla primigenia**. La preparación de datos es idéntica: el `canaritos.txt` de cada corrida A2 coincide byte a byte con el de su corrida original (20 de 20), así que A2 queda pareada con las otras tres. Los notebooks actuales ya incluyen A2 en el bucle.

---

## 3. Requisitos

- VM de Google Cloud creada desde el template `to-128ram-08vcpu` (8 vCPU, 128 GB de RAM). El pico de memoria está en la preparación de datos (~60–68 GB). Con menos RAM la corrida usa swap y se vuelve varias veces más lenta.
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
sed -i 's/semilla_primigenia <- [0-9]*/semilla_primigenia <- 186653/; s/experimento <- [0-9]*/experimento <- 10100/' P8_WF_Testing_Multimes_GrupoB_10100.ipynb
# verificar: tiene que mostrar la semilla, el experimento y seq(8000, 14000
grep -o 'semilla_primigenia <- [0-9]*\|experimento <- [0-9]*\|seq(8000, [0-9]*' P8_WF_Testing_Multimes_GrupoB_10100.ipynb
```

> Si ya existe `~/buckets/b1/exp/WF<experimento>`, el notebook **reutiliza** lo que encuentre (BO terminadas, modelos grabados). Para empezar de cero, usar otro número de experimento o renombrar esa carpeta.

### 4.2 Lanzar la corrida en segundo plano

Desde una terminal SSH de la VM (no desde la interfaz de Jupyter), parado en la carpeta del notebook:

```bash
setsid nohup jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.timeout=-1 --ExecutePreprocessor.kernel_name=ir \
  P8_WF_Testing_Multimes_GrupoB_10100.ipynb > log_10100.txt 2>&1 < /dev/null &
```

- `setsid nohup ... &`: la corrida sigue aunque se cierre la terminal. El aviso `[1]+ Done` que aparece después no significa que terminó.
- `--inplace`: el notebook queda guardado con todas las salidas al terminar.
- Mientras corre, no abrir el notebook en Jupyter.

### 4.3 Seguimiento

```bash
EXP=10100
W=~/buckets/b1/exp/WF$EXP
ls $W/BO_log_*.txt                                                         # BO terminadas
for d in AA A2 AB AC; do echo "$d: $(ls $W/modelos/$d 2>/dev/null | wc -l)/10"; done   # modelos finales
top -b -n 1 | head -8                                                      # proceso R activo
grep -i -c error log_$EXP.txt                                              # tiene que dar 0
tail -2 log_$EXP.txt                                                       # al final: "Writing ... bytes to ..."
```

Terminó cuando existe `resultado_opciones.txt` y el log cierra con `[NbConvertApp] Writing ... bytes to ...`.

**Tiempo por corrida:** ~15–22 h con 8 vCPU y las cuatro opciones (preparación ~35 min, cada BO ~1–2 h, cada final train de 10 modelos ~1,5–5 h según los hiperparámetros elegidos).

**Si la VM se cae:** relanzar el mismo comando. La preparación de datos se recalcula, pero las BO terminadas (`BO_log_<opción>.txt`), las BO a medias (`HT_<opción>.RDATA`) y los modelos ya grabados (`modelos/<opción>/`) se retoman.

**Si la VM aparece apagada:** puede ser una VM spot que Google apagó, o el apagado automático de la cátedra (~30 min después de que no queda nada procesando, por ejemplo tras un error). Antes de relanzar, mirar `grep -i error log_<experimento>.txt`.

---

## 5. Corridas realizadas

### AA, AB y AC (experimentos 101xx)

| Grupo | Experimento | Semilla primigenia | Integrante |
|---|---|---|---|
| B | 10100 | 186653 | Lautaro Suarez |
| B | 10101 | 315779 | Lautaro Suarez |
| B | 10101 | 135007 | Carlos Veliz |
| B | 10102 | 631861 | Lautaro Suarez |
| B | 10102 | 250007 | Carlos Veliz |
| B | 10103 | 667657 | Lautaro Suarez |
| B | 10104 | 932227 | Lautaro Suarez |
| B | 10104 | 699967 | Lautaro Suarez |
| B | 10106 | 400009 | Lautaro Suarez |
| B | 10106 | 555557 | Genaro Virgilio |
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

### A2 (experimentos 201xx, misma semilla que la corrida original)

| Grupo | Experimento | Semilla primigenia | Notas |
|---|---|---|---|
| B | 20100 | 186653 | |
| B | 20101 | 315779 | |
| B | 20101 | 135007 | |
| B | 20102 | 631861 | |
| B | 20102 | 250007 | |
| B | 20103 | 667657 | |
| B | 20103 | 400009 | |
| B | 20104 | 932227 | |
| B | 20104 | 555557 | |
| B | 20105 | 699967 | |
| A | 20100 | 149911 | |
| A | 20101 | 234571 | |
| A | 20102 | 345679 | |
| A | 20103 | 567899 | |
| A | 20104 | 322633 | |
| A | 20106 | 999959 | |
| A | 20107 | 999953 | |
| A | 20108 | 999931 | |
| A | 20109 | 999979 | |
| A | 20110 | 104759 | |

> Corridas con el mismo número de experimento pertenecen a integrantes distintos y se ejecutaron en buckets distintos. Las carpetas de `resultados/` se llaman `analisis_<experimento>_Grupo_<grupo>_semilla_<semilla>`.

---

## 6. Salidas de cada corrida

En `~/buckets/b1/exp/WF<experimento>/`:

| Archivo | Contenido | En el repo |
|---|---|---|
| `resultado_opciones.txt` | Grupo B: ganancia en cada corte fijo (`gan_8000` … `gan_14000`) por opción. Grupo A: AUC de la BO (solo registro) | sí |
| `BO_log_<opción>.txt` | Todas las iteraciones de cada BO y sus hiperparámetros | sí |
| `PARAM.yml` | Parámetros, hiperparámetros finales y tiempos | sí |
| `curva_de_ganancia.pdf` | Grupo B: curvas de ganancia de las opciones | sí |
| `impo_<opción>.txt`, `canaritos.txt` | Importancia de variables | sí |
| `envios_kaggle.txt` | Grupo A: índice de los archivos enviados (opciones × 13 cortes) | sí |
| `kaggle_resultados_<exp>.csv`, `analisis_<exp>_Grupo_a_semilla_<semilla>.csv` | Grupo A: Public score de cada envío, bajado con la API de Kaggle | sí |
| `curva_<opción>.txt` | Grupo B: ganancia acumulada y suavizada cada 50 envíos (versión reducida de `ganancias_*.txt`) | sí |
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

1. Para cada corrida y opción se calcula **un solo valor**: la **ganancia máxima** de producción entre los 13 cortes fijos (8.000 a 14.000). Por indicación del profesor no se usa el promedio de la ganancia.
2. Por semilla se calculan las diferencias A2 − AA, AB − AA y AC − AA.
3. Se aplica el test de Wilcoxon de rangos con signo, pareado por semilla, y el test de Friedman con las cuatro opciones, **por grupo** (las escalas de A y B no son comparables):

```r
# tb: una fila por semilla, columnas AA, A2, AB, AC (ganancia máxima entre los 13 cortes)
wilcox.test(tb$A2, tb$AA, paired = TRUE, alternative = "greater")
wilcox.test(tb$AB, tb$AA, paired = TRUE, alternative = "greater")
wilcox.test(tb$AC, tb$AA, paired = TRUE, alternative = "greater")
friedman.test(as.matrix(tb[, c("AA", "A2", "AB", "AC")]))   # las cuatro opciones a la vez
```

---

## 8. Resultados

Ganancia máxima por semilla. Grupo B: ganancia local en 202107 (÷120.000). Grupo A: Public de Kaggle sobre 202109.

**Medias de los máximos**

| Grupo | AA (1 mes) | A2 (2 meses) | AB (3 meses) | AC (5 meses) |
|---|---|---|---|---|
| B (10 semillas) | 597,3 | 598,8 | 597,0 | 599,7 |
| A (10 semillas) | 448,7 | 449,9 | 445,2 | 447,2 |

**Tests**

| Grupo | Comparación | Semillas | Dif. media | A favor | p unilateral | p bilateral |
|---|---|---|---|---|---|---|
| B | A2 vs AA | 10 | +1,5 | 6/10 | 0,377 | 0,754 |
| B | AB vs AA | 10 | −0,4 | 5/10 | 0,615 | 0,846 |
| B | AC vs AA | 10 | +2,4 | 7/10 | 0,116 | 0,232 |
| B | Friedman (4 opciones) | 10 | | | | 0,644 |
| A | A2 vs AA | 10 | +1,3 | 6/10 | 0,385 | 0,770 |
| A | AB vs AA | 10 | −3,4 | 3/10 | 0,958 | 0,105 |
| A | AC vs AA | 10 | −1,4 | 3/10 | 0,784 | 0,492 |
| A | Friedman (4 opciones) | 10 | | | | 0,086 |

**Conclusión:** con 10 semillas por grupo, **ninguna ventana de testing mejora al baseline**. Las cuatro opciones quedan dentro de ~3 puntos en B y ~5 en A, sin una tendencia ordenada con la cantidad de meses (1 → 2 → 3 → 5). En A el Friedman da p = 0,086, sobre todo porque AB queda por debajo del resto; no llega al 5 % y no se repite en B (p = 0,644). La variabilidad entre semillas (la propia BO elige configuraciones muy distintas con la misma ventana) es mucho mayor que cualquier efecto de los meses de validación. En el grupo A el Public mide solo una parte de 202109: conviene repetir el análisis con el Private cuando cierre la competencia.
