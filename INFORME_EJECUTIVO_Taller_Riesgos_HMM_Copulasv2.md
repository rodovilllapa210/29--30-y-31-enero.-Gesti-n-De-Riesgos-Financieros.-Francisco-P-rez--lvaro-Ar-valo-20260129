# Informe Ejecutivo de Riesgos - Cartera Multi-Activo

**Fecha:** 2026-02-15  
**Periodo analizado (panel CORE):** precios 2007-04-11 a 2026-02-13; retornos 2007-04-12 a 2026-02-13  
**Marco metodologico:** HMM 2 estados + copulas por estado + simulacion Monte Carlo (10.000 trayectorias, 6 meses)

## 1) Decision ejecutiva

La cartera puede operar en regimen base, pero su riesgo de cola se deteriora de forma material cuando aparece estres de credito con contagio de correlaciones. La prioridad para Comite es mantener control condicional por regimen y gatillos tacticos de desriesgo.

**Juicio de riesgo (hoy):**
- Riesgo base: **Amarillo**
- Riesgo bajo shock de credito: **Rojo**
- Prioridad de accion: **Alta**

## 2) Evidencia clave para Comite

### 2.1 Riesgo marginal y calidad de defensivos

- HYG: la volatilidad diaria en Estres sube **+215.28%** frente a Normal (0.0132 vs 0.0042).
- GLD: clasificacion **Refugio no concluyente / mixto** (**2/4** criterios).
- Lectura: el bloque de credito amplifica cola izquierda y GLD no compensa de forma robusta en este corte.

### 2.2 Dependencia y perdida de diversificacion

- Correlacion media t-copula (off-diagonal): **0.225 -> 0.318** (Normal -> Estres).
- Dependencia de cola media lambda_L (off-diagonal): **0.043 -> 0.061**.
- t-df por estado: Normal=10.0, Estres=10.0.
- Lectura: en estres aumentan comovimiento y co-caidas extremas.

### 2.3 Calidad del simulador (base)

- Vol anualizada cartera: Real **0.179** vs Simulado **0.179** (gap -0.001).
- VaR99 diario: Real **-3.114%** vs Simulado **-3.078%** (gap +0.036 pp).
- ES99 diario: Real **-4.732%** vs Simulado **-4.278%** (gap +0.454 pp).
- % dias en Estres: Real **14.72%** vs Simulado **14.55%**.

### 2.4 Ajustes metodologicos de universo (ENPH / GME)

- **ENPH (inicio real en 2012):** se habilita reconstruccion controlada por *proxy splice* para preservar cobertura temporal del panel CORE. El proxy se selecciona con fallback automatico **FSLR -> TAN -> ICLN**.
- **GME (evento extremo 2021):** se aplica politica de control idiosincratico configurable (**exclude / winsorize / include**). En baseline ejecutivo se usa **exclude** para evitar sesgo en HMM, copulas y colas.
- **Lectura para Comite:** estos ajustes aumentan robustez temporal y reducen sensibilidad a outliers no representativos del riesgo estructural.

## 3) Escenarios de estres (horizonte 6 meses)

| Escenario | VaR99 6m | ES99 6m | Delta ES99 6m vs Base | Corr media estres |
|---|---:|---:|---:|---:|
| S2 Crisis Credito 2008 | -41.10% | -48.14% | -22.80% | 0.522 |
| S3 Liquidez Global | -36.07% | -44.63% | -19.29% | 0.449 |
| S1 Estanflacion 2022 | -28.65% | -35.98% | -10.64% | 0.430 |
| Base | -21.42% | -25.35% | +0.00% | 0.307 |

**Escenario dominante:** **S2 Crisis Credito 2008** (ES99 6m = -48.14%).

## 4) Reverse stress (romper umbral)

| Familia | Lambda minima | Estado |
|---|---:|---|
| S2 Crisis Credito 2008 | 0.50 | CROSS |
| S3 Liquidez Global | 0.70 | CROSS |
| S1 Estanflacion 2022 | 0.90 | CROSS |

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
- La reconstruccion de ENPH via proxy introduce riesgo de especificacion (mitigado con fallback y trazabilidad del ticker usado).
- La decision de excluir/winsorizar GME afecta metricas de cola; se recomienda reportar sensibilidad frente a `include`.
- El modelo de estado es parsimonioso y no incorpora todas las variables macro posibles.
- Se recomienda revision trimestral y backtesting continuo para mantener robustez operativa.
