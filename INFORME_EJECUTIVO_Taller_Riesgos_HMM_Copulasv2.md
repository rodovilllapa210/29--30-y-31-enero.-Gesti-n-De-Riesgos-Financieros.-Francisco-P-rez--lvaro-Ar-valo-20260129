# Informe Ejecutivo de Riesgos - Cartera Multi-Activo

**Fecha:** 2026-02-11  
**Periodo analizado (panel CORE):** 2007-04-12 a 2026-02-09  
**Marco metodologico:** HMM 2 estados + copulas por estado + simulacion Monte Carlo (10.000 trayectorias, 6 meses)

## 1) Decision ejecutiva

La cartera puede operar en condiciones normales, pero su perfil de cola se deteriora de forma relevante cuando aparece estres de credito y contagio de correlaciones. La prioridad para Comite es aprobar control condicional por regimen (no solo por riesgo promedio) y activar gatillos tacticos de desriesgo.

**Juicio de riesgo (hoy):**
- Riesgo base: **Amarillo**
- Riesgo bajo shock de credito: **Rojo**
- Prioridad de accion: **Alta**

## 2) Evidencia clave para Comite

### 2.1 Riesgo marginal y calidad de defensivos

- HYG: la volatilidad en Estres sube **170.14%** frente a Normal.
- GLD: clasificacion **No refugio robusto** (score=1/4 criterios).
- Lectura: el bloque de credito explica gran parte del incremento de cola y GLD no compensa de forma robusta en este corte.

### 2.2 Dependencia y perdida de diversificacion

- Correlacion media t-copula: **0.224 -> 0.306** (Normal -> Estres).
- Dependencia de cola media (lambda_L): **0.042 -> 0.105**.
- t-df por estado: Normal=10.0, Estres=7.0.
- Lectura: en estres aumentan simultaneamente comovimiento medio y co-caidas extremas.

### 2.3 Calidad del simulador (base)

- Vol anualizada cartera: Real **0.176** vs Simulado **0.180** (gap +0.004).
- VaR99 diario: Real **-3.197%** vs Simulado **-3.172%** (gap +0.025%).
- ES99 diario: Real **-4.655%** vs Simulado **-4.540%** (gap +0.115%).
- % dias en Estres: Real **14.16%** vs Simulado **14.06%**.

### 2.4 Ajustes metodologicos de universo (ENPH / GME)

- **ENPH (inicio real en 2012):** se habilita reconstruccion controlada por *proxy splice* para preservar cobertura temporal del panel CORE. El proxy se selecciona con fallback automatico **FSLR -> TAN -> ICLN**.
- **GME (evento extremo 2021):** se aplica politica de control idiosincratico configurable (**exclude / winsorize / include**). En baseline ejecutivo se usa **exclude** para evitar sesgo en HMM, copulas y colas.
- **Lectura para Comite:** estos ajustes aumentan robustez temporal y reducen sensibilidad a outliers no representativos del riesgo estructural.

## 3) Escenarios de estres (horizonte 6 meses)

| Escenario | VaR99 6m | ES99 6m | Delta ES99 6m vs Base | Corr media estres |
|---|---:|---:|---:|---:|
| S2 Crisis Credito 2008 | -46.25% | -54.58% | -25.33% | 0.505 |
| S3 Liquidez Global | -40.63% | -49.04% | -19.80% | 0.442 |
| S1 Estanflacion 2022 | -35.75% | -43.53% | -14.28% | 0.420 |
| Base | -23.62% | -29.24% | +0.00% | 0.296 |


**Escenario dominante:** **S2 Crisis Credito 2008** (ES99 6m = -54.58%).

## 4) Reverse stress (romper umbral)

| Familia | Lambda minima | Estado |
|---|---:|---|
| S2 Crisis Credito 2008 | 0.50 | CROSS |
| S3 Liquidez Global | 0.65 | CROSS |
| S1 Estanflacion 2022 | 0.95 | CROSS |


Interpretacion: el shock tipo credito requiere menor intensidad para romper el umbral, por lo que debe tratarse como riesgo prioritario de gestion.

## 5) Plan 30-60-90 (accionable)

### En 30 dias
1. Aprobar limites por regimen (ES99 diario y ES99 6m) y umbrales de alerta.
2. Activar cuadro diario de seguimiento: P(Estres), correlacion media en estres y consumo de limite.

### En 60 dias
1. Ejecutar rebalanceo tactico para reducir vulnerabilidad conjunta equity + HYG cuando se active alerta.
2. Definir cobertura defensiva alternativa a GLD como refugio principal.

### En 90 dias
1. Cerrar ciclo de gobierno de modelo: recalibracion HMM/copulas + validacion real-vs-sim.
2. Reportar a Comite KPIs de eficacia: reduccion de ES99 y menor sensibilidad a shocks de credito.

## 6) Limitaciones y control

- Los resultados dependen de la calibracion historica del panel CORE.
- La reconstruccion de ENPH via proxy introduce riesgo de especificacion (se mitiga con fallback y trazabilidad del ticker usado).
- La decision de excluir/winsorizar GME afecta metricas de cola; se recomienda reportar sensibilidad frente a `include`.
- El modelo de estado es parsimonioso y no incorpora todas las variables macro posibles.
- Se recomienda revision trimestral y backtesting continuo para mantener robustez operativa.
