# Taller de Gestión de Riesgos Financieros (HMM, Cópulas y Stress Testing)

Notebook técnico para modelar cambios de régimen, dependencia y riesgo de cola en una cartera multi-activo long-only.

## Objetivo

Responder, con evidencia cuantitativa:

- Cómo cambia el riesgo de la cartera al entrar en crisis.
- Qué activos pierden diversificación bajo estrés.
- Qué intensidad mínima de shock rompe umbrales de riesgo (reverse stress).

## Universo de activos

- **Acciones USA:** AAPL, AMZN, BAC, BRK-B, CVX, ENPH, GME, GOOGL, JNJ, JPM, MSFT, NVDA, PG, XOM
- **Bonos USA:** IEF (7-10Y), SHY (1-3Y)
- **Crédito:** HYG
- **Refugio:** GLD

Periodo objetivo: desde `2006-01-01` hasta fecha disponible.

## Ajustes metodológicos clave (ENPH / GME)

- **ENPH:** como su histórico real empieza en 2012, se permite reconstrucción por *proxy splice* para preservar cobertura temporal del panel CORE.
  - Fallback automático de proxy: `FSLR -> TAN -> ICLN`.
  - Se escala el proxy en la primera fecha de solape y desde el listing real se usa ENPH observado.
- **GME:** se aplica política configurable para controlar sesgo idiosincrático extremo:
  - `exclude` (baseline recomendado),
  - `winsorize`,
  - `include`.

## Metodología por fases

1. **Fase 1 (HMM):** identificación de regímenes Normal vs Estrés.
2. **Fase 2:** riesgo marginal por estado (media, vol, skew, kurtosis, VaR/ES).
3. **Fase 3:** dependencia por estado (correlación, cola empírica y cópulas gaussiana/t).
4. **Fase 4:** Monte Carlo con cambio de régimen (10.000 trayectorias, 6 meses).
5. **Fase 5:** stress testing por escenarios.
6. **Fase 6:** reverse stress (mínimo shock que cruza umbral).

## Estructura del proyecto

```text
├── Taller_Riesgos_HMM_Copulasv2_main.ipynb
├── INFORME_EJECUTIVO_Taller_Riesgos_HMM_Copulasv2.md
├── Taller_Riesgos_HMM_Copulasv2_analisis.md
├── outputs_taller/
│   ├── prices_adj_close.csv
│   ├── returns_log.csv
│   ├── phase2_risk_by_state.csv
│   ├── phase3_copula_summary.json
│   ├── phase4_*.csv
│   ├── phase5_*.csv
│   └── phase6_*.csv
├── requirements.txt
└── README.md
```

## Instalación y ejecución

1. Crear entorno virtual (recomendado):

```bash
python -m venv venv
venv\Scripts\activate
```

2. Instalar dependencias:

```bash
pip install -r requirements.txt
```

3. Ejecutar notebook:

```bash
python -m jupyter notebook Taller_Riesgos_HMM_Copulasv2_main.ipynb
```

## Características técnicas

- Detección automática de ruta de proyecto (`TALLER_DIR` opcional).
- Caché local en `outputs_taller/`.
- Semillas fijas para reproducibilidad.
- Modo HMM real-time con filtrado forward (sin look-ahead).
- Checklist automático de entregables al final del notebook.

## Limitaciones

- Dependencia de calibración histórica del panel CORE.
- Riesgo de especificación en ENPH cuando se usa proxy.
- Sensibilidad de colas al tratamiento de GME (`exclude/winsorize/include`).
- Necesidad de recalibración y backtesting periódico.

## Uso

Proyecto académico para análisis de riesgo de mercado y comunicación a Comité de Riesgos.
