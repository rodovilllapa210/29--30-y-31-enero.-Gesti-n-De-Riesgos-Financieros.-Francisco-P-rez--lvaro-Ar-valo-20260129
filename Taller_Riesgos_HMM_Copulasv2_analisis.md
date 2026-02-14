# Taller de Gestión de Riesgos (HMM + Cópulas + Stress Testing)

Este documento es una guía para resolver el taller del enunciado Taller_Riesgos_HMM_Copulasv2.

La idea central: construir un motor de stress testing condicionado al estado del mercado. No basta con simular retornos “bonitos”; hay que capturar:

1) no estacionariedad (regímenes),
2) cambios en las distribuciones marginales (colas, asimetría),
3) cambios en la dependencia (correlación que sube y dependencia en cola), y
4) traducirlo a impacto en una cartera long-only.

---

## 0) Lo que pide el enunciado (y cómo te lo van a evaluar)

### Contexto profesional
Eres Riesgos de Mercado en una gestora multi-activo y presentas al Comité de Riesgos.

### Objetivo
Desarrollar en Python un sistema capaz de contestar:

- ¿Cómo cambia el riesgo de la cartera al entrar en crisis?
- ¿Qué activos dejan de diversificar bajo estrés?
- ¿Qué deterioro mínimo hace “inaceptable” el perfil actual? (reverse stress)

### Universo (ETFs)
- Acciones USA: AAPL, AMZN, BAC, BRK-B, CVX, ENPH, GME, GOOGL, JNJ, JPM, MSFT, NVDA, PG, XOM
- IEF (Treasuries USA medio, proxy 7–10Y)
- SHY (Treasuries USA corto, proxy 1–3Y)
- HYG (High yield)
- GLD (Oro)

### Restricciones
- Cartera long-only
- Pesos suman 1

### Entregables
1) Notebook reproducible con análisis + gráficos + simulaciones.
2) Informe ejecutivo (1–2 páginas) para Comité.

### Criterios de evaluación (traducción práctica)
Te puntúan por:

- Coherencia económica: que lo que detectas tenga sentido (crisis = más vol, colas peores, dependencia sube).
- Interpretación del riesgo: no solo números; qué significa para la cartera.
- Comunicación: gráficos correctos y mensaje claro.
- Realismo profesional: evitar “matemática bonita” sin credibilidad.


---

## 1) Estructura recomendada del Notebook

Una estructura:

1. Setup y datos: descarga, limpieza, retornos.
2. Exploratorio: stats globales y primeros gráficos.
3. Fase 1 (HMM): estimación de estados + validación.
4. Fase 2 (marginales por estado): colas y asimetrías.
5. Fase 3 (dependencia): correlación vs cópulas y cola.
6. Fase 4 (escenarios sintéticos): simulación y validación.
7. Fase 5 (peor que crisis): diseño de shocks + impacto.
8. Fase 6 (reverse stress): pérdida crítica y mínimos shocks.
9. Conclusiones (para el informe ejecutivo).

---

## 1b) Diccionario de tecnicismos

### Retornos, percentiles y colas

- **Retorno logarítmico**: $r_t=\log(P_t/P_{t-1})$.
  - Intuición: mide el cambio “en proporción” de un día a otro. Para cambios pequeños, $r_t$ se parece al % diario.
  - Por qué se usa: suma bien en el tiempo. Si tienes varios días, $\sum r_t = \log(P_T/P_0)$.
  - Mini‑ejemplo: si $P$ pasa de 100 a 101, entonces $r\approx\log(1.01)\approx 0.995\%$.

- **Percentil / cuantil**: un percentil es un **umbral**.
  - Percentil 5% (cuantil 0.05) = “el nivel por debajo del cual cae aproximadamente el 5% de los días”.
  - Mini‑ejemplo: si el cuantil 5% de la cartera es −2%, significa que “1 de cada 20 días (aprox.) la cartera pierde 2% o más”.

- **Cola izquierda**: la parte “mala” de la distribución (pérdidas grandes, retornos muy negativos).
  - Por qué importa: muchas carteras se ven parecidas “en el centro” (días normales), pero se diferencian en la cola (días de crisis).
  - Traducción comité: “no me preocupa el día típico; me preocupa el peor 1–5%”.

### Restricciones de cartera

- **Long‑only**: no permites posiciones cortas (pesos negativos).
  - En pesos: $w_i\ge 0$ y $\sum_i w_i=1$.
  - Intuición: repartes el 100% del capital entre ETFs; no puedes “apostar a la baja” ni apalancarte vía shorts.
  - Por qué importa en riesgos: limita el tipo de coberturas que puedes construir (las coberturas deben venir de activos defensivos, no de cortos).
  - **Ejemplo**: cartera equiponderada en 8 ETFs ⇒ $w_i=1/8=0.125$ para cada uno; si AAPL cae −3% y MSFT −4%, el retorno de cartera ese día es $0.125\times(-3\%)+0.125\times(-4\%)+\dots$ (suma ponderada de los 8 retornos).

### VaR vs ES (por qué usamos ES)

- **VaR** a nivel $\alpha$ (ej. 5%): el **cuantil** de la distribución de retornos de cartera.
  - Lectura: “umbral de pérdida” a ese nivel.
  - Ejemplo: $VaR_{5\%}=-2\%$ ⇒ “en el 5% peor de los días, la pérdida es de al menos ~2%”.

- **ES** a nivel $\alpha$ (Expected Shortfall): el **promedio** de retornos en el peor $\alpha\%$.
  - Lectura: “cuando me va mal de verdad, ¿qué tan mal va, de media?”.
  - Ejemplo: $ES_{5\%}=-4\%$ ⇒ “promedio de pérdidas dentro del peor 5% ≈ 4%”.

Mini‑ejemplo (intuición):
- Si tus 5 peores días (de 100) fueron: −2%, −2.5%, −3%, −6%, −10%.
- $VaR_{5\%}$ está cerca de −2% (el “umbral”).
- $ES_{5\%}$ estaría cerca de −4.7% (la media de esos 5 días).

Lectura: VaR te dice “hasta dónde suele llegar”; ES te dice “cómo de horrible es, de media, cuando te va mal de verdad”.

Comparación clave (muy útil en este taller):
- Si dos escenarios tienen VaR parecido pero ES muy distinto, significa que **los extremos dentro de la cola** son mucho peores en uno de ellos.

### Estadísticos típicos (Fase 2)

- **Volatilidad**: desviación típica de retornos (dispersión).
  - Intuición: “qué tanto se mueve” el activo/cartera en un día típico.

- **Skew (asimetría)**: si los extremos negativos y positivos son igual de probables.
  - Skew negativo (típico en risk‑on): más “sustos” hacia abajo que hacia arriba.
  - Traducción comité: “hay más probabilidad de caídas grandes que de subidas grandes”.
  - **Ejemplo**: AAPL en Estrés con skew −0.8 ⇒ la distribución tiene una “cola izquierda” más larga; los peores días son más extremos (en %) que los mejores.

- **Kurtosis (curtosis)**: cuánta masa hay en colas (extremos) vs una normal.
  - Kurtosis alta ⇒ más días muy extremos (buenos o malos) de lo que predeciría una campana.
  - Traducción comité: “más ‘días de película’ de lo que sugiere un modelo normal”.
  - **Ejemplo**: kurtosis 8 (frente a 0 para una normal) ⇒ muchos más días con movimientos >2–3% de lo que predeciría una campana; típico en renta variable en crisis.

- **QQ‑plot**: compara cuantiles de tus datos vs cuantiles de una distribución (p.ej. normal).
  - Si en la cola izquierda los puntos se van muy por debajo de la línea, sugiere colas más pesadas (pérdidas extremas más probables).

### Regímenes, filtrado y suavizado

- **Régimen / estado**: “modo” del mercado (Normal vs Estrés). Se modela como variable oculta $S_t$.
  - Intuición: el mercado cambia de comportamiento (volatilidad, colas, dependencia) y esos cambios duran semanas/meses.

- **Probabilidad filtrada** $P(S_t=\text{crisis}\mid X_{1:t})$:
  - Es una probabilidad “usable en tiempo real”: solo usa información hasta el día $t$.
  - Se interpreta como un semáforo: 0.1 = “casi seguro normal”, 0.9 = “casi seguro estrés”.

- **Viterbi**:
  - Devuelve una **etiqueta por día** (Normal/Estrés) que maximiza la probabilidad total de la secuencia.
  - Útil para “dibujar bloques” (episodios) sin estar hablando de probabilidades cada día.

- **EWMA** (suavizado exponencial):
  - Reduce el “serrucho” de una serie (por ejemplo, $P(\text{crisis})$) y da una señal más estable.

- **Histéresis**:
  - Entras en crisis con un umbral alto (p.ej. 0.7) y sales con uno más bajo (p.ej. 0.3).
  - Intuición: “cuesta entrar, pero también cuesta salir”, lo cual evita cambios espurios.

### Dependencia: correlación, cola y cópulas

- **Correlación (Pearson)**: dependencia lineal “en el centro”.
  - Qué contesta: “en un día típico, ¿se mueven en la misma dirección?”
  - Por qué puede fallar en crisis: porque resume con **un solo número** lo que pasa en toda la distribución. Dos activos pueden tener correlación 0.3 “en promedio” y, aun así, caer juntos en los peores días.
  - Ejemplo mental: si casi siempre se mueven poco y de forma independiente, pero en 5 días de pánico caen a la vez muy fuerte, la correlación puede no “gritar” ese riesgo.

- **Dependencia en cola** ($\lambda_L$): dependencia “en días muy malos”.
  - Qué contesta (frase comité): “cuando el mercado se hunde, ¿se hunden juntos?”
  - Traducción concreta: “si AAPL está en su peor 5%, ¿MSFT también suele estar en su peor 5%?”
  - Mini‑ejemplo numérico (con $q=5\%$):
    - Tienes 1.000 días.
    - “Día malo AAPL” = AAPL en su peor 5% ⇒ ~50 días.
    - “Día malo MSFT” = MSFT en su peor 5% ⇒ ~50 días.
    - Ambos malos a la vez ocurren 20 días.
    - Entonces $\mathbb{P}(\text{ambos en cola})=20/1000=0.02$ y
      $$\lambda_L(5\%)=\frac{0.02}{0.05}=0.4$$
    - Lectura: 0.4 significa “en la cola, caen juntos bastante más de lo que te diría independencia pura”.
  - Por qué es comité‑friendly: habla directamente de **diversificación que falla** (no es un tecnicismo por postureo, es “riesgo de caída conjunta”).

- **Cópula**: herramienta para separar dos capas:
  1) **marginales**: “qué tan malo puede ser AAPL (o la cesta equity) por sí solo” (colas, skew, etc.),
  2) **dependencia**: “si AAPL tiene un día malo, ¿qué pasa con el resto ese mismo día?”.

  Analogía útil:
  - Marginal = “catálogo de días” de cada activo.
  - Cópula = “regla de sincronización” que decide si esos días malos llegan a la vez.

- **Pseudo‑observaciones**: convertir cada retorno en su **percentil** dentro del régimen: $U\in(0,1)$.
  - Qué estás haciendo realmente: cambias “unidades” (%, volatilidad distinta por activo) por “posición en su ranking”.
  - Mini‑ejemplo:
    - Si para AAPL un retorno −3% es un percentil 1% en Estrés, entonces lo representas como $U_{AAPL}=0.01$.
    - Si para IEF un retorno −1% es un percentil 1% (porque IEF se mueve menos), lo representas también como $U_{IEF}=0.01$.
    - Ahora puedes hablar de dependencia entre “días muy malos” sin mezclar escalas.
  - Ventaja: la cópula modela “cómo de sincronizados están esos percentiles” de forma consistente entre activos.

### t‑cópula, grados de libertad y “colas conjuntas”
**Ejemplo sencillo:** con $\nu=30$ la cola conjunta es suave; con $\nu=4$ aumentan los dias en que varios activos caen a la vez. Es pasar de ?malos dias aislados? a ?dias malos sincronizados?.


- **t‑cópula**: cópula diseñada para que existan más casos de “varios malos a la vez”.
  - Idea: en un pánico real hay un “factor común” (liquidez/ventas forzadas/margen) que empeora todo simultáneamente.
  - La t‑cópula reproduce esto mejor que la gaussiana porque añade “colas” en la parte conjunta.

- **Grados de libertad** $\nu$ (df): “control de brutalidad” de la cola.
  - Intuición: cuanto menor es $\nu$, más probables son shocks grandes (y por tanto percentiles extremos simultáneos).
  - $\nu$ pequeño (3–6) ⇒ más días tipo “todo rojo”.
  - $\nu$ grande (30+) ⇒ el modelo se comporta casi como gaussiano.

- **Comparación clave (Gauss vs t)**:
  - Gaussiana: puede clavar la correlación media, pero tiende a subestimar la **dependencia en cola**.
  - t‑cópula: conserva la correlación media y además permite que, en la cola, el conjunto se mueva más “en bloque”.
  - Por qué esto afecta tanto al ES: ES promedia lo peor; si los peores escenarios son “muchos caen juntos”, ES empeora mucho.

### PSD (matrices válidas) y por qué aparece en el código
**Ejemplo sencillo:** si fuerzas correlaciones incompatibles (A-B muy negativo, A-C muy positivo y B-C muy positivo a la vez), la matriz deja de ser coherente. La proyeccion PSD corrige eso sin cambiar la narrativa del shock.


- **PSD (positive semidefinite)**: condición para que una matriz de correlación sea “coherente”.
  - Traducción llana: no puedes asignar correlaciones arbitrarias par a par; todas juntas tienen que ser compatibles.
  - Si no es PSD, el modelo implicaría que alguna combinación de activos tiene varianza negativa (imposible).

- Por qué importa: para simular retornos correlacionados hacemos una descomposición tipo **Cholesky**.
  - Si la matriz no es PSD, Cholesky falla (porque el modelo matemático no tiene sentido).

- Por qué se rompe en stress: en Fase 5/6 “empujas” muchos pares hacia valores extremos.
  - Ejemplo realista: subir risky‑risky a 0.95 *y además* forzar ciertos pares bonos‑equity hacia +0.2 puede crear incompatibilidades globales.

- Solución: proyectar a la “más cercana PSD”.
  - Qué significa: “la ajusto lo mínimo posible para que sea válida”.
  - Lectura comité: no cambia la narrativa (“más contagio”), solo evita una incoherencia técnica.

### Monte Carlo (simulación) y error

- **Simulación Monte Carlo**: estimar VaR/ES con muchos escenarios simulados.
  - En vez de una sola cifra “predicha”, generas una distribución de P&L con miles de posibles mañanas.
  - Ventaja: te permite medir cola (VaR/ES) sin asumir una normalidad perfecta.

- **Error Monte Carlo**: con menos escenarios, VaR/ES sale más ruidoso.
  - Intuición: con pocas tiradas el percentil 1% depende mucho de “qué te tocó”; con muchas, se estabiliza.
  - Mini‑ejemplo: con 10.000 escenarios, el 1% peor son solo 100 escenarios; si subes a 50.000, ya son 500 y el ES1 se vuelve más estable.
  - Regla práctica: si ves curvas (ES vs $\lambda$) con “dientes de sierra”, sube `n` o promedia varias semillas.

### EVT (mencionado como alternativa)

**Ejemplo simple:** si solo tienes 5 a?os de datos pero quieres estimar un evento ?1 de cada 1.000 d?as?, EVT te permite extrapolar la cola con supuestos adicionales.

- **EVT (Extreme Value Theory)**: técnicas para modelar colas y extrapolar más allá del histórico.
  - Qué aporta: si quieres estimar eventos más raros que tu muestra (p.ej. 1 cada 1.000 días), EVT es un camino formal.
  - Qué cuesta: requiere supuestos adicionales (elección de umbral, estabilidad de cola) y validación.
  - Por qué no es imprescindible aquí: el enunciado del taller suele valorar más una narrativa + shocks defendibles (Fase 5) que una EVT perfecta sin interpretación económica.

Nota de numeración: en el notebook la sección puede aparecer como “8) Fase 6” por cómo se han insertado celdas; en este informe la llamamos “Fase 6” por el enunciado.

---

## 2) Fase 1 — Identificación de estados del mercado (HMM)

### Objetivo y salida (lo que entregas en esta fase)

En esta fase se identifican **regímenes latentes** del mercado de forma **dinámica** (usable en tiempo real). El deliverable mínimo para el Comité es:

- una serie diaria de probabilidad $P(\text{crisis})$,
- una segmentación en **bloques** (normal vs crisis/estrés) para un gráfico legible,
- evidencia de que el régimen “crisis/estrés” tiene interpretación económica y captura episodios conocidos.

En nuestro baseline trabajamos con **2 estados**:
- **normal**
- **crisis/estrés**

### Decisiones tomadas (baseline) y posibles mejoras

Esta subsección deja claro **qué decisiones hemos tomado** en el baseline (lo que está implementado) y **qué mejoras** se pueden explorar después como sensibilidad.

**Decisiones tomadas (baseline del notebook)**

1) **Número de estados $K$**
- Elegimos **$K=2$** (normal vs crisis/estrés) para maximizar interpretabilidad y estabilidad.

2) **Qué observa el modelo (señales/features)**
- Trabajamos con un conjunto **core** y defendible: retornos (`r_equity`, `r_hy`, `r_10y`, `r_2y`, `r_gld`), proxy de crédito (`r_credit = r_hy - r_10y`), volatilidad realizada 21d (`vol_equity_21`, `vol_hy_21`) y drawdown rolling 252d (`dd_equity`).
- Todas las señales son **observables en tiempo real** y se construyen a frecuencia diaria.

3) **Cómo modelamos las señales dentro de cada estado**
- Usamos una **gaussiana por estado** (por practicidad y por disponibilidad de librería) y estandarizamos con `StandardScaler`.
- Para robustez, usamos covarianza `diag` y un prior de persistencia en la matriz de transición (evita cambios espurios día-a-día).

4) **Frecuencia**
- Diario con retornos logarítmicos es estándar para vigilancia y régimen.

**Posibles mejoras (sensibilidades) — no incluidas en el baseline**

- Probar **$K=3$** si aparece un estado intermedio (“tensión”) estable e interpretable.
- Sustituir la gaussiana por **t-Student por estado** (colas) si se implementa/dispone de librería adecuada.
- Ampliar señales con medidas de **co-movimiento** (correlaciones rolling), **asimetría** (downside vol) o **tipos/curva** (spreads), y validar si mejora estabilidad sin sobreajustar.

### Modelo HMM (explicación clara para defenderlo)

Un HMM es una forma ordenada de decir: *“el mercado no se comporta siempre igual; alterna regímenes que duran un tiempo, y cada régimen genera patrones distintos en nuestras señales”*.

**1) Variable de régimen (oculta) y persistencia**

Llamamos $S_t$ al régimen en el día $t$ (no se observa directamente). El supuesto Markoviano significa que el régimen de hoy depende principalmente del de ayer:

$$P(S_t=j \mid S_{t-1}=i) = A_{ij}$$

Ejemplo (2 estados: 0=Normal, 1=Estrés). Una matriz de transición típica y fácil de interpretar sería:

| De \ A | Normal (0) | Estrés (1) |
|---|---:|---:|
| **Normal (0)** | 0.98 | 0.02 |
| **Estrés (1)** | 0.03 | 0.97 |

Lectura “modo Comité” (qué significa cada celda):
- **0.98 (Normal→Normal)**: si ayer era normal, lo habitual es seguir normal.
- **0.02 (Normal→Estrés)**: pasar de normal a estrés es relativamente raro (cuando ocurre, suele ser porque cambian varias señales a la vez, no por un día aislado).
- **0.97 (Estrés→Estrés)**: si entras en estrés, tiende a durar (inercia).
- **0.03 (Estrés→Normal)**: volver a normal ocurre, pero no suele ser inmediato.

Intuición (sin fórmulas):
- $S_t$ es una etiqueta que el modelo “imagina” para cada día (p.ej. **normal** o **estrés**). No la observas directamente (no existe una columna “hoy es crisis”), por eso es un **estado oculto**.
- La suposición markoviana dice: para estimar el estado de hoy, lo más importante es el estado de ayer. No significa que el pasado no importe; significa que se resume en “ayer”.

Qué es $A$ (la matriz de transición):
- $A_{ij} = P(S_t=j \mid S_{t-1}=i)$ se lee: “si ayer estaba en $i$, ¿qué probabilidad hay de que hoy esté en $j$?”
- Con 2 estados (0=Normal, 1=Estrés):
  - $A_{00}$: Normal→Normal (seguir normal)
  - $A_{01}$: Normal→Estrés (entrar en estrés)
  - $A_{10}$: Estrés→Normal (salir de estrés)
  - $A_{11}$: Estrés→Estrés (seguir en estrés)

Por qué “diagonal alta” = inercia:
- La diagonal de $A$ son $A_{00}$ y $A_{11}$ (quedarte en el mismo estado). Si son altas (p.ej. 0.97–0.99), el modelo está diciendo que los regímenes **duran**.
- Esto evita que un solo día raro fuerce un cambio: para cambiar de estado, el patrón de señales $X_t$ tiene que parecerse de forma consistente al “perfil” del otro estado.

Interpretación para Comité:
- Si la diagonal de $A$ es alta, hay **inercia**: una crisis no aparece/desaparece “por un día raro”.
- Los cambios de régimen se interpretan como **cambios estructurales** del entorno de riesgo (volatilidad, crédito, drawdown, etc.).

Qué significa “cambio estructural” (en este contexto):
- No es “hoy la equity cayó y ya está”. Es que durante un tramo de tiempo, varias señales cambian de forma coherente respecto al patrón normal (p.ej. sube `vol_equity_21`, empeora `r_credit`, el `dd_equity` se hace más profundo, etc.).
- El HMM captura esto porque compara qué “perfil” (normal vs estrés) explica mejor el vector completo $X_t$ y, además, la matriz $A$ favorece la persistencia: para cambiar de estado suele requerirse que ese patrón se sostenga más de un día.

**2) Qué “ve” el modelo (señales) y qué asume dentro de cada estado**

Cada día resumimos el mercado con un vector $X_t$ (retornos, volatilidades, drawdown, crédito). La idea del HMM es:

- **El estado $S_t$** dice “en qué modo está el mercado” (normal vs estrés).
- **Las señales $X_t$** tienden a tomar valores distintos según ese modo.

En términos estadísticos, el modelo asume que, *condicionado al estado*, las señales se distribuyen de forma estable (con un “perfil” propio de cada régimen):

$$X_t \mid S_t=k \sim \mathcal{N}(\mu_k, \Sigma_k)$$

Interpretación:

- $\mu_k$ resume los **niveles típicos** del régimen (p.ej., en estrés suele haber `vol_equity_21` más alta, `dd_equity` más negativo y `r_credit` más débil).
- $\Sigma_k$ resume la **variabilidad** de las señales dentro del régimen (qué tanto “se mueve” cada señal alrededor de su nivel típico; con covarianza `diag` lo tratamos de forma robusta, sin forzar co-movimientos complejos).

En lenguaje llano: “si hoy fuera un día de crisis, esperaríamos ver un patrón de señales parecido a otros días de crisis; si fuera normal, parecido a otros días normales”. El HMM usa esa diferencia de patrones para inferir $P(S_t=\text{crisis})$.

**3) Cómo se identifica “crisis” sin imponer reglas a mano**

El modelo no etiqueta “crisis” por un umbral fijo de una feature. Primero estima dos (o $K$) perfiles. Después, *tú etiquetas* cuál es el estado “crisis/estrés” mirando su interpretación económica (por ejemplo: peor media de la cesta equity, mayor volatilidad, drawdown más profundo, crédito más deteriorado). Eso es defendible porque es una regla de lectura, no una regla de detección.

**4) Qué produce el notebook (outputs que importan)**

- **Señal continua**: una probabilidad diaria $P(S_t=\text{crisis}\mid X_{1:t})$ para monitorización.
- **Segmentación en episodios**: una etiqueta/secuencia de regímenes para sombrear bloques y poder hablar de “episodios de estrés” con fechas.

El punto clave: el HMM separa *incertidumbre diaria* (probabilidad) de *narrativa de episodios* (bloques), que es exactamente lo que un Comité necesita para discutir riesgos.

### Features usadas en el notebook (definición y qué aportan)

Nota: en el notebook las variables `r_*` son **retornos logarítmicos diarios**.

**Señales (núcleo del notebook)**

| Feature | Qué es | Qué aporta para detectar estrés |
|---|---|---|
| `r_equity` | Retorno diario (log) de la cesta equity | Señal principal de risk-on/risk-off; en crisis suele ser negativa y más extrema |
| `r_hy` | Retorno diario (log) de HYG | Captura estrés en crédito/high yield (se deteriora en crisis de spreads) |
| `r_10y` | Retorno diario (log) de IEF | Captura shocks en duración media (a veces hedge, a veces falla) |
| `r_2y` | Retorno diario (log) de SHY | Señal de tramo corto y estrés de tipos | 
| `r_gld` | Retorno diario (log) de GLD | Refugio/hedge potencial; ayuda a diferenciar tipos de crisis |
| `r_credit = r_hy - r_10y` | Diferencial de retornos HYG vs IEF | Proxy de widening/narrowing de spreads de crédito (aversión al riesgo) |
| `vol_equity_21` | Volatilidad realizada 21 días de `r_equity` (std rolling) | Clustering de volatilidad y persistencia del estrés |
| `vol_hy_21` | Volatilidad realizada 21 días de `r_hy` | Estrés específico en crédito |
| `dd_equity` | $\log(P_{eq}) - \max_{252d}\log(P_{eq})$ (≤0) | Daño persistente vs “mal día” |

En conjunto, estas features aportan dirección (retornos), intensidad (volatilidad), persistencia (drawdown) y mecanismo (crédito).

### Del HMM al gráfico (probabilidad vs bloques)
**Metafora rapida:** la probabilidad es como un termometro continuo; los bloques son *zonas de calor/frio* para comunicar al Comite cuando oficialmente consideras crisis.


Es importante separar dos objetos:

1) **$P(\text{crisis})$**: señal continua (0–1) útil para vigilancia.
2) **Bloques sombreado normal/crisis**: capa de presentación para que el Comité vea episodios claros.

En nuestro notebook, los “bloques” se obtienen aplicando reglas sobre $P(\text{crisis})$ (suavizado, histéresis, mínimos de duración). Estos umbrales son **post-procesado para comunicar**, no una regla fija sobre las features.

### Atribución por episodio: “drivers” (qué cambió más)
**Ejemplo simple:** si en un bloque (p.ej. 2008) los mayores z-scores son `vol_equity_21` alto y `r_credit` muy negativo, el mensaje es ?volatilidad se dispara? y ?credito se deteriora? en ese episodio.


Los “drivers” (o “atributos”) son una forma de explicar, para cada bloque de estrés/crisis, qué señales fueron las más “anómalas” comparadas con un día normal.

- Primero el HMM te da una probabilidad diaria $P(\text{crisis})$ y tú conviertes eso en bloques (episodios continuos de estrés).
- Para cada bloque, calculamos la media de cada feature dentro del bloque y la comparamos con la media de esas mismas features en días normales.
- Esa comparación se resume con un z-score:

$$z = \frac{\mu_{\text{bloque}} - \mu_{\text{normal}}}{\sigma_{\text{normal}}}$$

- $z>0$: la señal está más alta de lo normal (ej. `vol_equity_21` alta).
- $z<0$: más baja de lo normal (ej. `r_equity` muy negativo).
- $|z|$ grande: esa señal destaca como “driver” del episodio.

Importante: no prueba causalidad (“esto causó la crisis”). Sirve para justificar ante comité: “el modelo marcó este episodio porque en ese tramo vimos un patrón consistente de volatilidad alta / drawdown profundo / crédito deteriorado, etc.”

### Ejemplo completo (paso a paso, con intuición)

Piensa la Fase 1 como un clasificador probabilístico con memoria:

1) Cada día construimos $X_t$ (features). Ejemplo de día con estrés:
- `r_equity` muy negativo,
- `vol_equity_21` alta,
- `dd_equity` muy negativo,
- `r_credit` negativo.

2) El HMM compara qué perfil (normal vs crisis) explica mejor ese $X_t$ y devuelve $P(\text{crisis})$.

3) Si la probabilidad se mantiene alta, el post-procesado genera un bloque de crisis/estrés (sombreado), evitando “código de barras”.

4) Para explicar el bloque, se calculan drivers (top z-scores) y se reportan en una tabla.

**Ejemplo numérico (un día y un bloque):**
- **Un día típico de estrés** (valores ilustrativos): `r_equity` = −1.8%, `vol_equity_21` = 2.2%, `dd_equity` = −12%, `r_credit` = −0.5%. El HMM compara este vector con los perfiles Normal/Estrés y puede devolver $P(\text{crisis})=0.92$ → ese día se sombrea como crisis.
- **Drivers de un bloque** (ejemplo de tabla): si en un episodio 2008–2009 los z-scores son `dd_equity: z=−5.9`, `vol_hy_21: z=+2.7`, `vol_equity_21: z=+1.7`, `r_equity: z=−0.1`, la lectura es: "el modelo marcó este episodio sobre todo por drawdown muy profundo y volatilidad alta (equity y crédito); el retorno medio de la cesta equity dentro del bloque no es el que más destaca en valor absoluto".

### Validación (lo que hay que demostrar al Comité)

- Interpretación económica: crisis/estrés debe mostrar mayor volatilidad, peores retornos y/o señales de crédito deterioradas.
- Episodios conocidos: deben aparecer periodos de estrés relevantes (p.ej. 2008/2020) sin forzar fechas.
- Estabilidad: el régimen no debe alternar cada pocos días (evitar “flicker”).

### Receta de parámetros (baseline replicable)

Para que el notebook sea defendible y repetible, fija una “config baseline” (y solo la cambias si justificas por qué):

- **Regímenes**: $K=2$ (normal/crisis).
- **Features** (todas diarias):
  - retornos: `r_equity, r_hy, r_10y, r_2y, r_gld`,
  - proxy crédito: `r_credit = r_hy - r_10y`,
  - volatilidad realizada: ventana 21d (`vol_equity_21`, `vol_hy_21`),
  - drawdown: rolling peak 252d (mín. 60 obs) para `dd_equity`.
- **Escalado**: `StandardScaler` (media 0, var 1).
- **HMM**: `GaussianHMM` con `covariance_type="diag"`, `n_iter=500`, `tol=1e-4`, `random_state=42`.
- **Persistencia**: prior de transiciones fuerte en la diagonal (p.ej. diag=200 y off-diag=1). Esperable ver probabilidades tipo $P(s\to s)\approx 0.97–0.99$.
- **Identificación de “crisis”**: el estado con peor `equity_mean` y mayor `vol_equity_21` (y drawdown más negativo / peor `credit_mean`).
- **Visualización**:
  - suavizado de $P(\text{crisis})$ con EWMA (en el main: `SMOOTH_SPAN=60`),
  - sombreado por Viterbi con limpieza: en el main `min_true=60` días y `max_false_gap=10` (bloques más “macro”; si prefieres episodios más cortos, puedes usar 15 y 5).

Si quieres un régimen más “macro” (menos señales), mantén o sube `min_true` (p. ej. 60); para más sensibilidad, baja a 15–30 y/o añade histéresis (entrar crisis con $P>0.7$ y salir con $P<0.3$).

---

## 3) Fase 2 — Riesgo marginal por estado

### Qué te piden
Comparar la distribución de retornos de cada ETF entre regímenes: media, volatilidad, asimetría, colas.

En nuestro baseline, esto se hace **condicionando por el estado** detectado en Fase 1 (Normal vs Estrés) y reportando medidas de centro y cola por activo.

### Cómo explicarlo
Piensa en dos capas:

1) “Centro” de la distribución:
- Media/mediana por estado.

2) “Riesgo” y “extremos”:
- Volatilidad por estado.
- Cuantiles extremos (p. ej. 1% y 5%): ¿cuánto puede caer en un día/semana en crisis?
- Asimetría (skew) y curtosis (kurtosis) por estado.

### Qué métricas concretas reportar
- Por activo y por estado:
  - media, volatilidad,
  - VaR histórico a 5% y 1%,
  - ES histórico a 5% y 1%,
  - skew y kurt.

**Implementación en el notebook (baseline):**
- Tabla por activo y estado con `mean`, `vol`, `skew`, `kurt`, `VaR_5`, `ES_5`, `VaR_1`, `ES_1`.
- Gráfico rápido de comparación: barras de **ES(5%)** por activo en Normal vs Estrés.
- Export: se guarda la tabla completa en `outputs_taller/phase2_risk_by_state.csv`.

### Lectura de resultados (lo que le importa al Comité)
El Comité está preocupado por **comportamiento extremo en crisis**, así que aquí la métrica “estrella” es el **ES** (Expected Shortfall): no solo mira un cuantil (VaR), sino el **promedio de las peores pérdidas**.

En nuestros datos (ES al 5%, diario), el deterioro en Estrés vs Normal es más acusado en:
- **GME**, **ENPH** y **NVDA**: suelen concentrar los mayores deterioros de cola y volatilidad.
- **AMZN** y **BAC** también empeoran claramente en cola izquierda.
- **HYG** (high yield) empeora de forma relevante (cola y volatilidad se amplían).

Activos más defensivos en esta muestra:
- **IEF** y **SHY**: el empeoramiento en ES(5%) es pequeño comparado con renta variable/crédito.
- **GLD**: empeora en estrés, pero menos que los activos de riesgo.

Sobre “media vs riesgo”: la **media** en Estrés no necesariamente cae (puede incluir rebotes dentro del mismo bloque), pero **las colas sí se deterioran**; por eso VaR/ES son el foco cuando el objetivo es “¿qué pasa en lo peor de lo peor?”.

### Gráficos que “venden” bien
- Histogramas/densidades por estado (dos colores).
- Histograma/densidad por estado con líneas de **VaR(5%)** y **VaR(1%)**: visualiza el desplazamiento de la cola izquierda en crisis.
- QQ-plot contra normal (por estado) para evidenciar colas.
- Boxplots por estado.

**Ejemplo de lectura (tabla Fase 2, valores ilustrativos):**

| Activo | Estado  | mean (%) | vol (%) | ES_5 (%) |
|--------|--------|----------|---------|----------|
| AAPL   | Normal | 0.04     | 1.0     | −1.5     |
| AAPL   | Estrés | −0.05    | 2.2     | −3.2     |
| NVDA   | Normal | 0.02     | 1.3     | −1.8     |
| NVDA   | Estrés | −0.12    | 2.8     | −4.5     |
| IEF    | Normal | 0.02     | 0.5     | −0.4     |
| IEF    | Estrés | 0.01     | 0.7     | −0.6     |

Lectura: "En Estrés, la cola de AAPL empeora (ES_5 pasa de −1.5% a −3.2%); NVDA empeora mucho más (ΔES_5 ≈ −2.7%); IEF casi no cambia → es el tipo de activo que mantiene diversificación en crisis."

---

## 4) Fase 3 — Dependencia entre activos (más allá de correlación)

### Por qué la correlación no basta
La correlación mide dependencia lineal “en el centro” y puede fallar justo donde más importa: en las colas (crisis).

Lo que busca el Comité: ver si “diversificadores” (bonos, oro) dejan de diversificar, y si la cartera se vuelve un “todo junto cae”.

### Qué cambia respecto a Fase 2
En **Fase 2** mirabas a cada activo por separado (su media, volatilidad, VaR/ES…).
En **Fase 3** miras la relación entre activos (**dependencia**): cómo se mueven juntos.

En el notebook se analiza en tres niveles (alineados con el enunciado):

- **(3a) correlación** (dependencia lineal “media”).
- **(3b) cola empírica** (caídas conjuntas en días malos).
- **(3c) cópulas** (modelo de dependencia para simular escenarios en Fase 4).

### 3a) Correlaciones por estado (Normal vs Estrés)

#### Qué es “correlación”
La correlación (Pearson) mide si dos series se mueven juntas de forma “lineal”.

- Corr = **+1**: se mueven muy sincronizados (si AAPL sube, MSFT suele subir).
- Corr = **0**: no hay relación lineal clara.
- Corr = **−1**: se mueven al contrario (si AAPL sube, IEF suele bajar).

#### Qué haces en 3a
Calculas la matriz de correlaciones usando solo días:

- **Normal** (panel 1)
- **Estrés** (panel 2)

Y luego construyes el panel 3:

$$\Delta Corr = Corr(\text{Estrés}) - Corr(\text{Normal})$$

#### Cómo leer los 3 paneles (ejemplos)
Imagina el par **AAPL–MSFT**:

- Si en Normal tienes corr 0.85 y en Estrés 0.92 → $\Delta = +0.07$.
- Traducción: *“en crisis se mueven aún más juntos → diversificación peor dentro de renta variable”*.

Para un par “diversificador” tipo **AAPL–IEF**:

- Si en Normal corr = −0.20 y en Estrés corr = −0.30 → $\Delta = −0.10$.
- Traducción: *“en crisis el bono protege más (más anticorrelación)”*.

#### Por qué a veces parece “igual”
Porque la correlación es un promedio “del centro”: aunque cambien los extremos, el mapa general puede parecer similar.
Además, en esta muestra el máximo $|\Delta|$ observado es moderado (del orden de ~0.12), por lo que el panel de diferencias no “explota” salvo que uses una escala más estrecha.

### 3b) Dependencia en colas (empírica): “¿se hunden juntos en días malos?”

Aquí ya no preguntas “¿se mueven juntos en promedio?”, sino:

*“Cuando AAPL está en sus peores días, ¿MSFT también está en sus peores días?”*

#### Qué significa “cola”
La “cola izquierda” son los retornos muy negativos.

- $q=5\%$: “los peores 5% días” (≈ 1 de cada 20)
- $q=1\%$: “los peores 1% días” (≈ 1 de cada 100), más extremo

#### La métrica que calculas: $\lambda_L(q)$
En el notebook se usa:

$$
\lambda_L(q)=\frac{\mathbb{P}(R_i\le Q_i(q),\ R_j\le Q_j(q))}{q}
$$

Desglosado en castellano:

- $Q_i(q)$: el umbral del activo $i$ para el peor $q\%$.
  - Ejemplo: si $q=5\%$ y el cuantil 5% de AAPL es −2%, entonces “día malo AAPL” = retorno ≤ −2%.
- $\mathbb{P}(R_i\le Q_i(q), R_j\le Q_j(q))$: porcentaje de días donde **ambos** están en su peor $q\%$.
- Dividir por $q$ es para normalizar y poder comparar.

#### Ejemplo numérico simple
Supón que en 1000 días (q=5%):

- “día malo AAPL” ocurre ~50 días.
- “día malo MSFT” ocurre ~50 días.
- ambos malos a la vez ocurren 20 días.

Entonces:

- Probabilidad conjunta = 20/1000 = 0.02
- $\lambda_L(5\%) = 0.02 / 0.05 = 0.4$

Interpretación:

- 0.4 significa: *“cuando miro la cola del 5%, una parte importante de veces caen juntos”*.
- Si en Estrés $\lambda$ sube (por ejemplo 0.4 → 0.6), el mensaje es: en crisis aumentan las caídas conjuntas en cola.

#### Qué significan los heatmaps de 3b

- Heatmap “$\lambda_L$ Normal”: intensidad = qué tan conjunta es la cola en Normal.
- Heatmap “$\lambda_L$ Estrés”: lo mismo en crisis.
- Heatmap “$\Delta\lambda_L$”: rojo = aumenta la cola conjunta en crisis, azul = disminuye.

Esto suele ser más “Comité” que la correlación porque habla directamente de **eventos extremos**.

**Implementación (notebook):**
- Heatmaps de $\lambda_L$ empírico en Normal vs Estrés (para $q=5\%$ y $q=1\%$) y sus deltas.
- Export de matrices a:
  - `outputs_taller/phase3_taildep_empirical_normal_q5.csv`
  - `outputs_taller/phase3_taildep_empirical_estres_q5.csv`
  - `outputs_taller/phase3_taildep_empirical_normal_q1.csv`
  - `outputs_taller/phase3_taildep_empirical_estres_q1.csv`

### 3c) Cópulas (Gaussiana vs t): “modelo de dependencia para simular escenarios”

Aquí pasas de medir dependencia (3a–3b) a construir un **modelo de dependencia** que luego usarás en Fase 4 para generar escenarios.

#### Qué es una cópula (definición útil)
Una cópula separa:

- **Marginales**: cómo se comporta cada activo por sí solo (Fase 2: colas, skew, etc.).
- **Dependencia**: cómo se relacionan entre sí (Fase 3).

Piensa así:

- Fase 2 te dice “cómo de mala puede ser la caída de AAPL”.
- Fase 3 con cópulas te dice “si AAPL cae, ¿MSFT cae a la vez? ¿y cuánto?”.

#### Paso 1: pseudo-observaciones (rangos)
Como cada activo tiene una distribución distinta, transformas cada serie a algo comparable:

- conviertes retornos a rangos → $U\in(0,1)$ (como “percentiles”).
- Un retorno muy malo → $U$ pequeño (cerca de 0.01).
- Un retorno “normal” → $U\approx 0.5$.

Esto permite modelar dependencia sin mezclar unidades/colas entre activos.

#### Cópula Gaussiana: qué significa el gráfico “Gauss copula corr”
La cópula gaussiana resume la dependencia con una matriz de correlación en el espacio transformado.

- Sirve para ver estructura general (muy parecido a correlación).
- Limitación clave: **no genera dependencia de cola fuerte** (asintóticamente la dependencia de cola tiende a 0).

Por eso el panel gaussiano se suele parecer mucho al de correlación y los deltas suelen ser “suaves”.

#### ¿Es coherente que Normal y Estrés se vean “casi iguales”?
Sí, y de hecho es un resultado típico cuando:

- El cambio de dependencia “media/lineal” entre regímenes es **moderado** (en tu ejecución, el propio panel de diferencias reporta un máximo $|\Delta|$ relativamente pequeño).
- La estructura global del universo se mantiene (p.ej. bloque de renta variable moviéndose bastante junto, y bloque de bonos moviéndose bastante junto), incluso aunque en crisis empeoren los extremos.

Interpretación práctica:

- Los heatmaps de **nivel** (Normal vs Estrés) te dicen “cómo es la dependencia en general”.
- El panel **$\Delta$** es el que contesta “qué cambia al entrar en crisis”.

Además, en t-cópula el mapa de cola asintótica depende sobre todo de (i) la matriz de correlación estimada y (ii) los grados de libertad $\nu$; si $\nu$ sale parecido en Normal y Estrés (en tu caso, del orden de 7 en ambos), es coherente que los mapas de nivel también se parezcan y el mensaje esté concentrado en el $\Delta$.

#### t-cópula: por qué importa
La t-cópula añade “colas más gordas” y permite dependencia más fuerte en extremos.

Tiene un parámetro $\nu$ (grados de libertad):

- $\nu$ pequeño (ej. 3–6): colas muy pesadas.
- $\nu$ grande: se parece más a gaussiana.

En este caso, la estimación produce un $\nu$ del orden de 7 en ambos estados (según el resumen exportado por el notebook), que es coherente con colas más pesadas que la gaussiana.

#### El heatmap “t-copula $\lambda_L$ (asint.)”
Ese heatmap no es el “q=1% histórico” de 3b; es la dependencia de cola **asintótica** que implica el modelo t.

Interpretación:

- valores más altos = el modelo cree que hay más probabilidad de caídas conjuntas extremas.
- $\Delta$ positivo en Estrés = *“en crisis la cola conjunta aumenta”*.

**Implementación (notebook):**
- Convertimos retornos a pseudo-observaciones $U\in(0,1)$ y ajustamos por estado:
  - **Cópula Gaussiana** (dependencia “tipo correlación”, cola asintótica 0).
  - **t-cópula** (permite dependencia en colas; parámetro $\nu$).
- Ajuste por pseudo log-likelihood (grid sobre $\nu$) y export:
  - `outputs_taller/phase3_copula_gaussian_corr_normal.csv`
  - `outputs_taller/phase3_copula_gaussian_corr_estres.csv`
  - `outputs_taller/phase3_copula_t_corr_normal.csv`
  - `outputs_taller/phase3_copula_t_corr_estres.csv`
  - `outputs_taller/phase3_copula_t_taildep_lambdaL_normal.csv`
  - `outputs_taller/phase3_copula_t_taildep_lambdaL_estres.csv`
  - resumen: `outputs_taller/phase3_copula_summary.json`

### Cómo lo contaría en el informe (texto simple)
“En Fase 3 analizamos la dependencia entre activos por régimen. La correlación cambia moderadamente entre Normal y Estrés, pero al mirar eventos extremos (cola izquierda) observamos que varios pares de activos de riesgo incrementan su probabilidad de caer simultáneamente en crisis. Para modelar esta dependencia de forma utilizable en simulación, ajustamos cópulas por estado (Gaussiana y t), donde la t-cópula permite capturar dependencia en cola, coherente con la preocupación del Comité por pérdidas conjuntas en crisis.”

Lectura rápida típica (la que “vende”): en Estrés sube la dependencia (y especialmente la cola izquierda) entre activos de riesgo: (AAPL, MSFT), (NVDA, AMZN), (AAPL, HYG), etc.; mientras que los defensivos (IEF/SHY/GLD) tienden a mantener o mejorar diversificación según el par.

### Cómo demostrar “fallo de diversificación”
- Compara correlación media en crisis vs normal.
- Mide tail dependence (si lo implementas): cuando un activo cae mucho, ¿otro tiende a caer también?
- Ejemplos concretos: (AAPL, MSFT), (AAPL, HYG), (IEF, AAPL), (GLD, AAPL).

Checklist fase 3:
- [x] Correlaciones por estado.
- [x] Medida/visual de dependencia en cola (empírica $\lambda_L$ y t-cópula).
- [x] Conclusión accionable: qué deja de diversificar (riesgo se mueve más “en bloque” en Estrés).

---

## 5) Fase 4 — Generación de escenarios sintéticos

### Qué estás haciendo realmente (en una frase)
Construyes un **generador de retornos diarios conjuntos** condicionado al régimen (Normal vs Estrés):

1) la **dependencia** (cómo caen juntos) la pones con una **t‑cópula por estado** (Fase 3c),
2) las **marginales** (colas/asimetría de cada activo) las pones con la **distribución empírica por estado** (Fase 2),
3) con eso produces muchos escenarios $R_{t+1}$ de 1 día y calculas VaR/ES de cartera.

Esto es exactamente lo que un Comité entiende por “simulación plausible”: no inventas una normal multivariante por comodidad, sino que **respetas colas y co‑movimientos observados** (y separados por régimen).

### Versión sin tecnicismos (para entenderlo al 100%)
Piensa que estás construyendo un “generador de días” para cada régimen. Ese generador tiene **dos piezas**: una decide *qué le puede pasar a cada activo* y la otra decide *si les pasa a la vez*.

**Pieza A — “Distribución empírica” = usar el histórico como catálogo de días**
Aquí no hay magia: para cada activo y régimen coges sus retornos históricos y los usas como referencia.

- Hazte la imagen mental: tienes una lista de retornos de AAPL **solo en Estrés**.
- Ordenas esa lista de peor a mejor.
- Si el simulador pide el percentil **0.02**, está pidiendo “un día tan malo como el **2% peor** de esa lista”.

Mini‑ejemplo:
- Si tienes 1000 días de AAPL en Estrés, el 2% peor son ~20 días.
- El percentil 0.02 sería “uno de esos ~20 peores días”.
- Si esos días rondan −4%, cuando pida 0.02, AAPL te saldrá alrededor de −4%.

**Qué ganas con esto:** que cada activo, por separado, tenga colas y asimetrías realistas *del régimen* (no una campana inventada).

**Pieza B — “t‑cópula” = la regla de ‘van juntos’**
Ahora viene lo importante para crisis: no solo importa que AAPL pueda caer −4%, sino si **cuando AAPL cae fuerte también caen fuerte MSFT/NVDA/HYG**.

Una forma simple de verlo:
- cada día el simulador genera un “percentil” para cada ETF (un número entre 0 y 1),
- percentil bajo (0.01–0.05) = “día malo” para ese activo,
- la cópula decide si esos percentiles salen **independientes** o **sincronizados**.

La **t‑cópula** es útil porque facilita que aparezcan días donde varios percentiles son muy bajos *a la vez* (extremos conjuntos), que es lo típico que rompe diversificación.

**¿Y $\nu$? (grados de libertad) — el ‘control de brutalidad’**
$\nu$ controla cuán frecuentes/intensos son esos extremos conjuntos:

- $\nu$ pequeño (p.ej. 3–6): más “días de película” con varios activos en percentiles muy bajos simultáneamente.
- $\nu$ grande (p.ej. 30+): comportamiento más “normalito”; sigue habiendo relación, pero es menos habitual ver extremos conjuntos tan fuertes.

**Frase para memorizar:**
- Empírica = “cada activo cae/sube como en su histórico del régimen”.
- t‑cópula = “en Estrés es más probable que varios estén en mal percentil a la vez”.
- $\nu$ = “cuánto permito esos malos días simultáneos (más pequeño = más)”.

Con esas dos piezas, los escenarios ya sirven para lo importante: simular P&L y calcular VaR/ES de cartera por régimen.

### Por qué hace falta (intuición)
Si simulas con una normal multivariante:

- las colas suelen salir demasiado “benignas”,
- y los eventos de caída conjunta severa (cola izquierda conjunta) quedan infraestimados.

La t‑cópula corrige la parte conjunta (dependencia en cola), y la inversa empírica asegura que **cada activo** mantenga su forma (asimetría/colas) dentro de cada régimen.

### Inputs de Fase 4 (qué reutilizas de fases previas)
Para cada estado $s\in\{\text{Normal},\text{Estrés}\}$ necesitas:

- Serie histórica de retornos por activo filtrada a ese estado: $\{r_{t,i}: S_t=s\}$.
- Parámetros de t‑cópula del estado (Fase 3c):
  - matriz de correlación $\rho_s$ (en el espacio de la cópula),
  - grados de libertad $\nu_s$ (controla grosor de colas conjuntas).

En el notebook, la Fase 4 (horizonte 1 día) genera dos “nubes” de escenarios: una para Normal y otra para Estrés.

### Paso a paso (lo que hace el notebook, versión defendible)

#### Paso 1 — Simular variables uniformes dependientes $U$ (la cópula)
El objetivo es obtener un vector $U=(U_1,\dots,U_d)$ con $U_i\in(0,1)$ que preserve la dependencia del estado.

En una **t‑cópula**, esto se consigue así:

1) Genera un vector gaussiano correlacionado $Z\sim\mathcal{N}(0,\rho_s)$.
2) Genera un factor de escala $W\sim\chi^2_{\nu_s}/\nu_s$.
3) Construye una t multivariante: $X = Z/\sqrt{W}$.
4) Convierte componente a componente a uniformes con la CDF de t:

$$U_i = F_{t,\nu_s}(X_i)$$

Interpretación:
- $\rho_s$ manda en el “patrón” de co‑movimiento.
- $\nu_s$ manda en cuán probable es ver eventos extremos conjuntos.

#### Paso 2 — “Poner” las marginales reales por estado (inversa empírica)
Una vez tienes $U_i$ como percentil, lo transformas a retorno con la inversa de la CDF marginal del activo $i$ en ese estado.

En versión empírica:

$$r_i^{\text{sim}} = \hat F^{-1}_{i,s}(U_i)$$

donde $\hat F^{-1}_{i,s}$ es el cuantil empírico construido con los retornos históricos del activo $i$ **solo en días del estado $s$**.

Consecuencia importante (muy buena para defensa):
- si miras el activo $i$ por separado, la distribución simulada reproduce (casi exactamente) la distribución histórica del estado, porque estás usando cuantiles empíricos.

#### Paso 3 — Construir retornos de cartera y métricas (VaR/ES)
Con un vector de pesos long‑only $w$ (suma 1), el retorno de cartera por escenario es:

$$r_p^{\text{sim}} = \sum_i w_i\, r_i^{\text{sim}}$$

Y con la muestra de escenarios (p.ej. 50.000), estimas:

- $VaR_\alpha$: cuantil $\alpha$ de $r_p$ (cola izquierda).
- $ES_\alpha$: media de $r_p$ condicionado a estar por debajo del $VaR_\alpha$.

**Ejemplo de un escenario simulado (1 día):** La t-cópula genera 8 percentiles $U_i$; si son todos bajos (p. ej. 0.02, 0.03, 0.01, …) es un "día de pánico". La inversa empírica convierte cada $U_i$ en retorno del activo $i$ en ese régimen (AAPL → −3.1%, MSFT → −3.8%, IEF → +0.5%, …). Con pesos equiponderados, $r_p = (1/8)\sum_i r_i^{\text{sim}}$. Repitiendo 50.000 veces obtienes la distribución de $r_p$ y de ahí VaR/ES.

En el notebook se reporta VaR/ES al 5% y 1% para Normal y Estrés, lo que permite contestar directamente: “¿cuánto empeora el riesgo de cartera cuando estás en crisis?”

### Cómo interpretar la validación (qué debe salir “bien”)
La validación compara (por estado) **Histórico vs Simulado** en dos niveles:

1) **Marginal** (por activo): media, vol, VaR/ES. Si falla aquí, tu inversa marginal o tu filtrado por estado no está bien.
2) **Dependencia** (entre activos): correlaciones Hist vs Sim (y su $\Delta$). Si falla aquí, el ajuste de la cópula (o la implementación del muestreo t‑cópula) no está reproduciendo el patrón conjunto.

Qué es aceptable:
- pequeñas diferencias en correlación (Monte Carlo + tamaño muestral por estado),
- pero no deberían aparecer “patrones nuevos” (p.ej. que bonos y equity pasen a correlación +0.8 si históricamente no es así).

### “¿Por qué hay dos simulaciones (Normal y Estrés)?”
Porque el objetivo del taller no es una simulación única “promedio”, sino un motor condicionado a régimen.

- En **Normal** esperas colas más benignas y/o dependencia menos dañina.
- En **Estrés** esperas colas peores y, sobre todo, más caídas conjuntas.

De hecho, aunque $\rho$ cambie “poco” (Fase 3a), el riesgo de cartera puede cambiar “mucho” porque:
- las colas marginales empeoran (Fase 2),
- y la dependencia en cola (t‑cópula) amplifica pérdidas conjuntas.

### Limitaciones (para decirlo con honestidad profesional)
- **Inversa empírica**: no extrapola más allá de lo observado; para “peor que crisis” (Fase 5) tendrás que forzar colas más pesadas (p.ej. escalado de cuantiles, EVT, o marginal paramétrica) y/o empeorar dependencia.
- **Muestra por estado**: si el estado Estrés tiene pocos días, algunos parámetros y cuantiles serán ruidosos; por eso es clave reportar y validar.
- **Horizonte 1 día**: aquí no modelas path‑dependence ni acumulación multi‑día; eso se añade si simulas una secuencia de estados con la matriz de transición del HMM (extensión natural).

Checklist fase 4 (lo mínimo que te puntúan):
- [x] Simulación reproducible (seed) y $N$ escenarios suficiente (decenas de miles).
- [x] Validación Hist vs Sim (marginal + correlación) por estado.
- [x] Traducción a impacto en cartera: VaR/ES en Normal vs Estrés.

---

## 6) Fase 5 — Stress testing “peor que crisis”

### Para qué sirve (mensaje Comité)
Fase 4 te dice *“si mañana fuera un día típico de Normal o de Estrés, ¿qué pérdidas esperaría?”*. Eso es útil, pero tiene una limitación: **solo reproduce lo que has visto** (o algo muy parecido).

Fase 5 sirve para responder a la pregunta profesional clave:

> “¿Qué pasa si ocurre algo peor que lo histórico y me falla la diversificación?”

Es decir: conviertes el modelo en un motor de **stress testing** y no solo de “simulación plausible”.

Esto se usa para:
- fijar límites (VaR/ES bajo stress, no solo en histórico),
- identificar vulnerabilidades (qué supuestos rompen la cartera),
- justificar coberturas o cambios de pesos con escenarios creíbles.

### Qué te piden exactamente
Ir más allá de lo observado históricamente, pero con narrativa económica plausible. Esto es clave: si lo haces “porque sí”, baja nota.

En otras palabras: no es “multiplico todo por 3” sin explicación; es diseñar shocks que el Comité reconoce como plausibles:
- **contagio sistémico** (todo riesgo cae junto),
- **inflación / tipos** (bonos dejan de cubrir),
- **crédito** (HYG se comporta como equity y su cola empeora).

### Cómo diseñar shocks (receta defendible)

Define 2–3 escenarios con:

1) Shock a medias (drift): empeoran rentabilidades esperadas.
2) Shock a volatilidad: multiplicador de sigma (por ejemplo x1.5 o x2).
3) Shock a dependencia: sube correlación / sube tail dependence.

Ejemplos del enunciado y cómo aterrizarlos:

- Contagio sistémico:
  - equity y crédito caen juntos; correlación sube.
  - HYG se comporta más como equity.

- Shock inflacionario (bonos dejan de cubrir):
  - IEF/SHY caen o su correlación con equity se vuelve menos negativa.

- Deterioro severo del crédito:
  - HYG tiene colas más gordas; dependencia cola con equity aumenta.

### Qué “mueves” realmente (traducción técnica a decisiones)
Un stress “peor que crisis” suele tocar 1–3 palancas:

1) **Dependencia peor**: sube la sincronización entre activos de riesgo (equity + crédito), o se reduce el hedge bonos/equity.
2) **Cola más pesada**: los días malos son más malos (se amplifica la cola izquierda).
3) **Volatilidad mayor**: dispersión general de retornos (por ejemplo, vol ×1.5).

Lo importante: que cada palanca tenga narrativa (mecanismo) y que cuantifiques el impacto en cartera.

### Profundizando: palancas, qué significan y cómo se aplican
Para “pasar de concepto a implementación” es útil separar 4 palancas (las 3 anteriores + drift). En el notebook se aplican sobre el **modelo de Estrés** (Fase 4) para generar escenarios más severos.

#### Palanca 1 — Dependencia (quién cae con quién)
Qué representa: *contagio* y *fallo de diversificación*. En el mundo real se manifiesta como:
- equity y crédito cayendo a la vez,
- bonos dejando de cubrir (correlación bonos/equity sube hacia 0 o incluso positiva),
- bloques del universo moviéndose “en bloque”.

Cómo se implementa (en este notebook):
- Se modifica la matriz de correlación de la t‑cópula del estado Estrés ($C_t$) y luego se proyecta a una correlación válida (PSD).
- Dos patrones típicos:
  1) **Contagio sistémico**: empujar correlaciones entre activos de riesgo hacia un objetivo alto (p.ej. 0.85–0.95).
  2) **Bonos fallan**: empujar correlaciones equity–IEF / equity–SHY hacia 0 o positiva.

Guía para defender parámetros:
- El objetivo no es “acertar el número exacto”, sino que el shock sea coherente y deje un mensaje de riesgo.
- Un rango defendible para un stress “serio” es aumentar correlaciones risky‑risky en +0.10 a +0.30 frente a Estrés baseline (dependiendo de cómo estés ya en crisis).

Cómo leer el impacto:
- Si al subir dependencia, $ES_{1\%}$ de cartera empeora mucho, tu cartera está expuesta a **caídas conjuntas** (diversificación frágil).

#### Palanca 2 — Colas (qué tan extremo puede ser un día malo)
Qué representa: *colas más gordas* y “días de película”. En la práctica, incluso sin cambiar mucho la correlación media, las colas pueden empeorar por:
- gaps,
- iliquidez,
- ventas forzadas,
- ampliación de spreads.

Cómo se implementa (dos mecanismos complementarios):
1) **Más cola conjunta** en la t‑cópula bajando $\nu$ (grados de libertad).
  - $\nu$ más pequeño ⇒ más probabilidad de que varios activos estén simultáneamente en percentiles muy bajos.
2) **Amplificación directa de la cola izquierda** sobre retornos ya simulados:
  - se identifica un umbral (por ejemplo el cuantil 5%) y se “estira” lo que queda por debajo para que sea peor.

Guía defendible:
- Bajar $\nu$ hacia 4–6 suele ser una manera razonable de representar colas más pesadas sin romper el modelo.
- Amplificar la cola por un factor 1.3–1.8 (solo en cola) es defendible como “peor que crisis” y evita distorsionar todo el centro.

Cómo leer el impacto:
- Esta palanca suele pegar más al **ES** que al VaR, porque el ES mira el promedio de lo peor de lo peor.

#### Palanca 3 — Volatilidad (dispersión general)
Qué representa: *régimen de alta incertidumbre* (vol clustering, ampliación de rangos intradía, etc.).

Cómo se implementa:
- Se reescala la dispersión alrededor de la media: vol × 1.4, 1.6, etc.
- Es un shock “grueso”: empeora tanto días buenos como malos, pero combinado con cola hace el stress más realista.

Guía defendible:
- Multiplicadores 1.3–2.0 son típicos en stress; 1.4–1.6 suele ser una elección “fuerte pero razonable”.

#### Palanca 4 — Drift / medias (mala tendencia)
Qué representa: que el mercado no solo sea volátil, sino que tenga *sesgo negativo* (risk‑off persistente).

Cómo se implementa:
- Añadir un desplazamiento negativo a activos de riesgo (p.ej. −10 a −30 bps diarios) o aplicar drift negativo a factores.
- En el baseline del notebook se ha dejado en 0 para no mezclar mensajes, pero es una palanca muy defendible si el enunciado pide narrativa macro.

### Cómo se traduce esto a los 3 stresses implementados
En el notebook se construyen 3 escenarios con narrativa y palancas explícitas (todos sobre el estado Estrés):

1) **S1_contagio_sistemico**
  - Dependencia: risky‑risky empujado hacia correlaciones muy altas.
  - Colas: $\nu$ reducido + cola izquierda amplificada.
  - Vol: multiplicador alto en activos de riesgo.
  - Mensaje: “cuando todo cae junto, el ES se dispara”.

2) **S2_inflacion_bonos_fallan**
  - Dependencia: equity–IEF / equity–SHY (y pares similares) empujados hacia 0/+.
  - Vol + colas: moderados en todo el universo.
  - Mensaje: “el hedge de duración deja de funcionar; el stress viene por el fallo del diversificador”.

3) **S3_credit_blowout**
  - Dependencia: HYG más sincronizado con equity.
  - Cola: amplificación fuerte de la cola de HYG.
  - Vol: aumento focalizado.
  - Mensaje: “crédito se comporta como equity en crisis y empeora la cola”.

### Buenas prácticas (para que no parezca ‘shock porque sí’)
- Shockea pocas palancas por escenario (1–2 principales) para que el mensaje sea limpio.
- Mantén coherencia: si dices “bonos fallan”, debe verse en correlaciones bonos/equity y en pérdida de cola de cartera.
- Reporta siempre baseline de Estrés vs stress, y da una frase de implicación (límites/hedge/pesos).

### Apoyo del material de clase (BME)
En las diapositivas [tema 6 Gestion de riesgos/BME_Riesgo financieros(P).pdf](tema%206%20Gestion%20de%20riesgos/BME_Riesgo%20financieros(P).pdf) se remarca justo la motivación de esta fase:

- El **VaR por sí solo** no está pensado para capturar bien pérdidas bajo **circunstancias extremas**.
- Por eso se recomienda complementar los informes de riesgo con **stress testing** y **análisis de escenarios**.
- La receta que describe (a nivel conceptual) es la misma que aplicamos aquí: definir **movimientos extremos de factores de riesgo**, revalorar la cartera bajo el escenario extremo y analizar la pérdida potencial.

Esto encaja muy bien con el enfoque de nuestro notebook: Fase 4 como generador por régimen y Fase 5 como “capa” de shocks defendibles para ir más allá de lo observado.

### Qué devuelve (outputs) y cómo se lee
Para cada escenario, produces:

- Una **distribución simulada de P&L de cartera** (histograma / densidad) → te da intuición visual.
- Métricas comparables entre escenarios:
  - $VaR_{5\%}$ / $ES_{5\%}$ y $VaR_{1\%}$ / $ES_{1\%}$,
  - media y volatilidad (secundario; en stress el foco son colas).

Lectura típica:
- Si un escenario empeora mucho $ES_{1\%}$ respecto al baseline de Estrés, estás expuesto a **eventos extremos conjuntos**.
- Si un escenario “bonos fallan” empeora mucho, tu diversificación depende demasiado de duración.

### Implementación en este notebook (baseline)
En el notebook, Fase 5 parte del modelo de Estrés de Fase 4 y construye 3 escenarios defendibles (1 día, 50.000 simulaciones):

- **S1_contagio_sistemico**: risky‑risky más sincronizados + colas conjuntas más pesadas + vol↑.
- **S2_inflacion_bonos_fallan**: bonos cubren menos (correlación bonos‑equity menos negativa) + vol↑.
- **S3_credit_blowout**: HYG más “equity‑like” (más sincronía con equity) + cola izquierda de HYG amplificada.

Y exporta:
- `outputs_taller/phase5_stress_summary_1d_n50000.csv` (tabla resumen VaR/ES por escenario)
- `outputs_taller/phase5_<SCENARIO>_1d_n50000.csv` (escenarios por activo)

### Qué reportar para la cartera
- P&L simulado (distribución) y percentiles.
- VaR/ES de la cartera en cada stress.
- Conclusión: qué activo/hedge deja de funcionar y qué decisión tomarías (límites, hedge, reducir exposición, etc.).

Checklist fase 5:
- [x] 2–3 escenarios con narrativa.
- [x] Impacto cuantitativo en cartera (VaR/ES/percentiles).
- [ ] Contribución al riesgo por activo (opcional, mejora nota).
- [x] Conclusión: qué ajustarías (pesos, límites, coberturas).

---

## 7) Fase 6 — Reverse stress testing

### Idea
En lugar de “qué pasa si…”, preguntas: “¿qué tiene que pasar para romperme?”.

En nuestro notebook lo implementamos con una versión **práctica y defendible**: partimos del **baseline del régimen Estrés** (Fase 4/5) y aumentamos gradualmente la severidad de shocks tipo Fase 5 usando un parámetro de intensidad $\lambda\in[0,1]$.

### Qué estás haciendo realmente (en una frase)
Estás encontrando el **deterioro mínimo** (en colas/dependencia/volatilidad) que hace que una **métrica de pérdida de cartera** supere un umbral.

Esto es muy “de Comité” porque el output final no es “una simulación bonita”, sino una frase accionable:

> “Con este perfil, basta con *X* (p.ej. 65% de un contagio sistémico) para cruzar un ES(5%) inaceptable; por tanto proponemos límites/hedges.”

### Versión sin tecnicismos (para entenderlo al 100%)

1) Tú eliges una línea roja (umbral): *“si el ES(5%) diario es peor que −4%, ya no lo aceptamos”.*
2) Diseñas una “rueda” de severidad: 0 = estrés histórico tal cual, 1 = el stress peor‑que‑crisis.
3) Vas girando esa rueda (subes $\lambda$) y miras cuándo cruzas la línea roja.
4) El **primer** punto donde cruzas es tu *reverse stress*: el mínimo shock que te rompe.

### Pasos

1) Define pérdida crítica (umbral “inaceptable”).

Dos formas típicas (ambas válidas):

- **Umbral absoluto** (más claro para comité):
  - Ejemplo 1D: $ES_{5\%} \le -4\%$ o $VaR_{1\%}\le -6\%$.
- **Umbral relativo al baseline** (útil cuando no quieres pelear el número exacto):
  - Ejemplo: “inaceptable = 1.6× el ES(5%) del baseline de Estrés”.

En el notebook usamos la versión relativa por defecto (se puede cambiar a absoluta).

2) Define vector de shocks con parámetros.
- a) caída media equity,
- b) multiplicador de vol,
- c) subida de dependencia,
- d) shock bonos (cambios de signo en correlación), etc.

3) Busca el mínimo deterioro que alcanza la pérdida crítica.

En el baseline lo hacemos con **búsqueda en rejilla** (grid) sobre $\lambda$ por cada familia de shock (S1/S2/S3). Es simple, transparente y defendible.

Si quisieras afinar más, puedes:
- aumentar resolución (más puntos de $\lambda$),
- hacer un “refinement” local (p.ej. buscar entre 0.60 y 0.70 con pasos 0.01), o
- usar búsqueda binaria si la curva es (aprox.) monótona.

### Qué significa $\lambda$ (y por qué es útil)

- $\lambda=0$: **sin shock adicional** → cartera bajo el “modelo de Estrés” calibrado por histórico (t‑cópula + marginal empírica).
- $\lambda=1$: **stress completo** tipo Fase 5 (el escenario “peor que crisis” tal cual lo definimos).
- Valores intermedios: mezcla lineal de palancas:
  - dependencia: $C(\lambda)=(1-\lambda)C_{base}+\lambda C_{stress}$ (y luego se fuerza PSD),
  - colas: $\nu(\lambda)$ interpolado (menos $\nu$ = colas más gordas),
  - marginales: `vol_mult(λ)` y `tail_mult(λ)` interpolados.

Interpretación “modo Comité”: *“con un deterioro del X% del stress diseñado, ya cruzamos el umbral inaceptable”*.

### Por qué usamos ES (y cuándo usarías otra cosa)

- **ES** (Expected Shortfall) es la métrica más coherente cuando te importa *“lo peor de lo peor”* porque promedia pérdidas extremas, no solo un punto (VaR).
- **VaR** sirve para comunicar percentiles (“en el 95% de los días no perderemos más de…”), pero puede ser menos estable para describir extremos.

Recomendación rápida:
- Para comité: reporta **ES5 y ES1** (y como apoyo VaR5/VaR1).
- Si el enunciado sugiere “pérdida mensual”, puedes aproximar un horizonte mayor con:
  - simulación multi‑día (más correcto), o
  - escalado de volatilidad (más simple, pero menos defendible en colas).

### Outputs (lo que tienes que enseñar)

- `outputs_taller/phase6_reverse_stress_grid_1d_n*.csv`: tabla con la métrica (VaR/ES) para cada familia de shock y cada $\lambda$.
- `outputs_taller/phase6_reverse_stress_minimum_1d_n*.csv`: mínimo $\lambda$ por familia que cruza el umbral y el mejor global.
- `outputs_taller/phase6_reverse_stress_plot_1d_n*.png`: gráfico métrica vs $\lambda$ con línea de umbral.

En el main, la Fase 6 usa un número de escenarios distinto al de Fase 4/5 (p. ej. `n=25000`, por eso los archivos salen como `*_n25000.*`); es coherente para ahorrar tiempo en la rejilla de $\lambda$.

### Cómo leer el resultado (ejemplo de la ejecución actual)

En la ejecución del notebook (1 día, métrica objetivo **ES(5%)**, umbral = **1.6×** el ES(5%) base en Estrés):

- Baseline ES5 (Estr?s) ? **?1.80%**
- Umbral objetivo ES5 ? **?2.89%**
- El shock m?nimo para cruzar el umbral fue **S1_contagio_sistemico** con $\lambda\approx 0.80$.

**Ejemplo de cómo se ve la curva ES(5%) vs $\lambda$ (cualitativo):**
- $\lambda=0$: ES5 ≈ −2.48% (baseline Estrés).
- $\lambda=0.3$: ES5 ? ?2.16%; $\lambda=0.5$: ES5 ? ?2.47%; $\lambda=0.8$: ES5 ? ?2.90% (cruza la l?nea ?2.89%).
- Para S2 (bonos fallan) la curva puede cruzar más tarde (p. ej. $\lambda=1.0$); para S3 (crédito) puede no cruzar en [0,1] → "con esta cartera el driver del punto de rotura es el contagio, no el crédito aislado."

Lectura práctica:
- “Con un ~65% del shock de contagio sistémico (más sincronización risky‑risky + colas más pesadas + vol↑) ya alcanzamos un ES(5%) inaceptable”.
- En este baseline, el escenario de crédito puro (S3) no cruza el umbral en el rango $\lambda\in[0,1]$ (indica que, con esta cartera equiponderada, el driver dominante del ‘break’ es contagio/dependencia más que el blowout aislado de crédito).

### Qué conclusiones sacas (y cómo lo venderías)

**1) El “driver” del punto de rotura**

- Si el mínimo $\lambda$ lo da S1 (contagio), tu vulnerabilidad dominante es **fallo de diversificación**: “todo cae a la vez” + colas más pesadas.
- Si lo da S2 (bonos fallan), tu vulnerabilidad dominante es **dependencia de duración como hedge**.
- Si lo da S3 (crédito), tu vulnerabilidad dominante es **crédito / HY** (y su correlación con equity en cola).

**2) Acción de gestión (la parte que te puntúa)**

Un reverse stress no se queda en “sale 0.80”; se traduce a acción:

- Reducir exposición a los activos que más contribuyen en esos días (p.ej. bajar bloque risk‑on).
- Añadir hedge explícito (opciones / convexidad) si el problema es cola.
- Si el problema es “bonos fallan”, diversificar hedge (oro/quality) o limitar duración concentrada.
- Definir límites: “si $P(\text{crisis})>X$ o si ES condicionado sube, reducimos riesgo”.

### Limitaciones (para ser honesto y aún así defenderlo)

- **Depende del diseño de shocks**: reverse stress te dice “mínimo dentro de este conjunto de palancas”. Si no incluyes un tipo de crisis (p.ej. FX o liquidez), no lo encontrarás.
- **Ruido Monte Carlo**: la curva ES vs $\lambda$ no tiene por qué ser perfectamente suave si el número de simulaciones es pequeño. (Solución: más `n`, o fijar semilla, o promediar varias semillas.)
- **Linealidad de la mezcla**: interpolar linealmente es una aproximación. Es defendible como baseline, pero conviene mencionarlo.

### Checklist fase 6 (baseline)
- [x] Definí pérdida crítica (umbral relativo 1.6× en ES5 1D).
- [x] Encontré el $\lambda$ mínimo por familia y el mínimo global.
- [x] Saqué implicaciones claras (driver dominante + acción).

- [x] Interpretación de gestión:
- “Con este perfil, basta con X para perder Y: proponemos límites/hedges.”

---

## 8) Informe ejecutivo (1–2 páginas): plantilla recomendada

Un informe “de comité” debería responder en 5 bloques:

1) Resumen (5–7 líneas): qué construiste y para qué.
2) Regímenes: cuántos, cómo se interpretan, cuándo ocurren.
3) Qué cambia en crisis:
   - marginales (colas/vol),
   - dependencia (diversificación falla).
4) Stress peor que crisis: 2–3 escenarios y su impacto en cartera.
5) Reverse stress: umbral crítico y mínimos shocks; recomendaciones.

---

## 9) Errores típicos (y cómo evitarlos)

- Correlación constante: siempre compararla por estado.
- Simulación gaussiana: prohibida “de facto” por el enunciado (solo como baseline).
- HMM ex-post: hay que hablar de filtrado/estimación dinámica.
- Sin validación: siempre “histórico vs simulado”.
- Sin narrativa: cada stress debe tener historia económica.


