# Reporte Técnico de Estrategias de Despliegue y Remediación: TechMarket Orders

**Integrantes:** Jose Ignacio Jeremias Aranda Baeza & Compañero  
**Institución:** DUOC UC  
**Contexto:** Evaluación de Infraestructura Tecnológica (EKS + CI/CD)  

---

## 🔍 IL3.1: Identificación de Escenarios de Error por Estrategia

Basado en el historial de desarrollo del microservicio de órdenes, se identifican las siguientes vulnerabilidades y manifestaciones de error asociadas a cada enfoque de despliegue:

### 1. All-in-Once (Todo a la vez)
* **Mecanismo de Manifestación:** El tráfico completo se conmuta inmediatamente al nuevo set de contenedores. Si existe un error en variables de entorno o dependencias rotas, el 100% de los usuarios experimenta un fallo instantáneo (`HTTP 500` / `Connection Refused`).
* **Severidad:** Crítica. Requiere intervención manual urgente si no está automatizado.

### 2. Rolling Update (Actualización Progresiva - Configuración Base Actual)
* **Mecanismo de Manifestación:** Los Pods se reemplazan de forma escalonada según las directivas `maxSurge` y `maxUnavailable`. 
* **Escenario de Error Típico:** Errores de configuración de imagen (`ErrImagePull` / `ImagePullBackOff`) como los simulados en las pruebas, o fallas intermitentes de latencia donde el contenedor inicia pero no responde a tiempo. Las sondas `livenessProbe` y `readinessProbe` son las encargadas de interceptar esto en el clúster.

### 3. Canary (Canario - Evidenciado en commits #51 y #58)
* **Mecanismo de Manifestación:** Un porcentaje reducido de tráfico (ej. 10%) se enruta al nuevo despliegue.
* **Escenario de Error Típico:** Degradación sutil del rendimiento o fugas de memoria (Memory Leaks) que solo se activan bajo carga real, manifestándose como timeouts (`HTTP 504`) detectados por alarmas de Amazon CloudWatch en ese subconjunto de usuarios.

### 4. Blue-Green (Azul-Verde - Evidenciado en commits #48, #50 y #59)
* **Mecanismo de Manifestación:** Coexisten dos entornos idénticos. El switch se realiza a nivel de Service o balanceador (ALB).
* **Escenario de Error Típico:** Incompatibilidad de persistencia o errores de sincronización con la base de datos (Amazon RDS). Si la nueva versión (Green) corrompe el esquema de datos, el error se propaga hacia atrás invalidando la infraestructura del entorno viejo (Blue).

---

## 📊 IL3.2: Análisis de Impacto de las Estrategias de Remediación

A continuación, se evalúan las ventajas y limitaciones de las acciones correctivas aplicadas en entornos ágiles bajo métricas de MTTR (Tiempo Medio de Recuperación), Costo y Disponibilidad:

| Mecanismo | Impacto en MTTR | Costo Operativo | Disponibilidad del Servicio | Limitaciones Técnicas / Contexto Ágil |
| :--- | :--- | :--- | :--- | :--- |
| **Rollback** (`kubectl rollout undo`) | **Mínimo (Segundos)**. Reinvoca la réplica anterior ya cacheada en el nodo. | Bajo. No requiere infraestructura extra permanente. | **Alta**. El tráfico vuelve a los Pods sanos sin caída de servicio. | No soluciona problemas de datos corruptos en BD. Ideal para CI/CD ágil. |
| **Re-despliegue** (Full Re-deploy) | **Alto**. Requiere reconstruir el build o re-compilar el pipeline. | Medio. Consumo de minutos de runner y cómputo. | **Baja/Media**. Puede inducir downtime si se fuerza un reemplazo masivo. | Lento para incidentes críticos en producción. |
| **Hotfix** (Parche en caliente) | **Moderado a Alto**. Depende de la velocidad de codificación del equipo. | Alto. Horas de ingeniería bajo presión extrema. | **Variable**. El sistema opera degradado hasta que el parche se aprueba. | Alto riesgo de introducir nuevos bugs por falta de pruebas exhaustivas. |
| **Feature Toggle** (Banderas de código) | **Inmediato (Milisegundos)**. Desactivación por API o consola de control. | Bajo a Medio. Costo de la herramienta de flags (ej. LaunchDarkly). | **Máxima**. Mitiga la falla anulando solo la característica rota. | Añade deuda técnica y complejidad limpia al código fuente de la app. |

---

## 📐 IL3.3: Diseño de la Estrategia de Remediación Temprana

Para garantizar la continuidad del negocio en **TechMarket Orders**, se ha diseñado e implementado un circuito cerrado de remediación automatizada estructurado de la siguiente forma:
