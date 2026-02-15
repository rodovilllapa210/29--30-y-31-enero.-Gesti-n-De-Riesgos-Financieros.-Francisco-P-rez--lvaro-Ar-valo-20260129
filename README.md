# Taller de Gestión de Riesgos Financieros - HMM, Cópulas y Stress Testing

Notebook técnico para modelar cambios de régimen, dependencia y riesgo de cola en una cartera multi-activo long-only.

## 🎯 Objetivo

Desarrollar un sistema completo de gestión de riesgos que conteste:
- ¿Cómo cambia el riesgo de la cartera al entrar en crisis?
- ¿Qué activos dejan de diversificar bajo estres?
- ¿Qué deterioro mínimo hace "inaceptable" el perfil actual? (reverse stress)

## 📊 Universo de Activos

**ETFs analizados:**
- **Acciones USA:** AAPL, AMZN, BAC, BRK-B, CVX, ENPH, GME, GOOGL, JNJ, JPM, MSFT, NVDA, PG, XOM
- **Bonos USA:** IEF (7-10Y Treasury), SHY (1-3Y Treasury)  
- **Crédito:** HYG (High Yield)
- **Refugio:** GLD (Oro)

**Periodo de análisis:** 2006-01-01 hasta fecha disponible (panel CORE desde 2007-04-12)

## 🏗️ Estructura del Proyecto

```
├── Taller_Riesgos_HMM_Copulasv2_main.ipynb    # Notebook principal con análisis completo
├── INFORME_EJECUTIVO_Taller_Riesgos_HMM_Copulasv2.md  # Resumen para Comité de Riesgos
├── Taller_Riesgos_HMM_Copulasv2_analisis.md   # Guía técnica detallada
├── outputs_taller/                           # Resultados y cache de datos
│   ├── prices_adj_close.csv                  # Precios históricos cacheados
│   ├── returns_log.csv                       # Retornos logarítmicos
│   ├── phase2_risk_by_state.csv             # Riesgo marginal por régimen
│   ├── phase3_*.csv                          # Análisis de dependencia y cópulas
│   ├── phase4_*.csv                          # Validación de simulaciones
│   ├── phase5_*.csv                          # Escenarios de stress testing
│   └── phase6_*.csv                          # Reverse stress analysis
├── requirements.txt                          # Dependencias Python
└── README.md                                 # Este archivo
```

## 🔧 Metodología

### Fase 1: Identificación de Regímenes (HMM)
- **Modelo:** Hidden Markov Model con 2 estados (Normal vs Crisis/Estres)
- **Features:** Retornos, volatilidad realizada, drawdown, proxy de crédito
- **Salida:** Probabilidad diaria de crisis y segmentación en episodios

### Fase 2: Riesgo Marginal por Estado
- **Análisis:** Media, volatilidad, skew, kurtosis por régimen
- **Métricas clave:** VaR y Expected Shortfall (5% y 1%)
- **Focus:** Deterioro de colas en activos de riesgo vs defensivos

### Fase 3: Dependencia (Correlación vs Cópulas)
- **3a:** Correlaciones por estado (Normal vs Estres)
- **3b:** Dependencia de cola empírica (λ_L)
- **3c:** Cópulas t-Student para modelar dependencia extrema

### Fase 4: Simulación Monte Carlo
- **Escenarios:** 10,000 trayectorias a 6 meses
- **Validación:** Comparación estadísticas reales vs simuladas
- **Reproducción:** Fidelidad del modelo en regímenes

### Fase 5: Stress Testing
- **Escenarios:** Crisis Crédito 2008, Estanflación 2022, Liquidez Global
- **Métricas:** VaR/ES a 6 meses por escenario
- **Análisis:** Impacto y levers de riesgo

### Fase 6: Reverse Stress
- **Objetivo:** Mínimo shock que rompe umbrales críticos
- **Metodología:** Búsqueda sistemática sobre parámetros de cópula
- **Salida:** Intensidad mínima por familia de escenarios

## 🚀 Requisitos de Instalación

1. **Clonar el repositorio:**
```bash
git clone <repository-url>
cd <repository-directory>
```

2. **Crear entorno virtual (recomendado):**
```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
# o
venv\Scripts\activate     # Windows
```

3. **Instalar dependencias:**
```bash
pip install -r requirements.txt
```

4. **Ejecutar el notebook:**
```bash
jupyter notebook Taller_Riesgos_HMM_Copulasv2_main.ipynb
```

## 📋 Dependencias Principales

- **Python:** ≥3.8
- **Análisis de datos:** pandas, numpy
- **Finanzas:** yfinance (descarga de datos)
- **Machine Learning:** scikit-learn, hmmlearn
- **Estadística:** scipy
- **Visualización:** matplotlib, seaborn
- **Cópulas:** copulas (para modelado de dependencia)

## 💡 Características Técnicas

- **Reproducibilidad:** Semilla fija en simulaciones
- **Cache local:** Datos descargados se guardan para acelerar ejecuciones
- **Paths portables:** Detección automática de directorio de proyecto
- **Validación robusta:** Múltiples checks de calidad de datos
- **Modo real-time:** HMM con filtrado forward (no look-ahead)

## 📈 Resultados Clave

### Riesgo por Régimen
- **HYG:** Volatilidad en Estres sube 170% vs Normal
- **GLD:** Clasificado como "No refugio robusto" (1/4 criterios)
- **Dependencia:** Correlación media 0.224→0.306, cola λ_L 0.042→0.105

### Escenarios de Stress (6 meses)
| Escenario | VaR99 | ES99 | ΔES99 vs Base |
|-----------|-------|------|---------------|
| Crisis Crédito 2008 | -46.25% | -54.58% | -25.33% |
| Liquidez Global | -40.63% | -49.04% | -19.80% |
| Estanflación 2022 | -35.75% | -43.53% | -14.28% |

### Validación del Modelo
- **Volatilidad cartera:** Real 0.176 vs Simulado 0.180
- **VaR99 diario:** Real -3.197% vs Simulado -3.172%
- **ES99 diario:** Real -4.655% vs Simulado -4.540%

## 🎓 Uso Educativo

Este taller está diseñado para estudiantes de **Introducción a los Sistemas Financieros** y cubre:

- Modelado de regímenes de mercado con HMM
- Análisis de dependencia más allá de correlación
- Stress testing y reverse stress methodologies
- Comunicación de riesgos a nivel ejecutivo

## ⚠️ Limitaciones

- Los resultados dependen de la calibración histórica del panel CORE
- El modelo de estado es parsimonioso (2 estados)
- Se recomienda revisión trimestral y backtesting continuo
- Los escenarios son históricos y pueden no capturar riesgos no observados

## 📞 Contacto

**Autor:** Francisco Pérez Álvaro Arévalo  
**Curso:** Introducción a los Sistemas Financieros  
**Institución:** MIAX  
**Fecha:** Enero 2026

## 📄 Licencia

Este proyecto tiene fines educativos y de investigación. Uso bajo permiso del autor.
