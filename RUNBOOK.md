# Runbook estandarizado: modelado de plantillas y construcción dinámica de gemelos digitales complejos

Este runbook define un procedimiento **estandarizado y repetible** para modelar
plantillas (scenes, dashboards y mapeos) y para **simplificar, mediante software
dinámico**, la construcción de gemelos digitales industriales complejos sobre
AWS IoT TwinMaker, AWS IoT SiteWise y Grafana.

El objetivo es que cualquier integrador pueda pasar de un conjunto de plantillas
parametrizadas a un gemelo digital funcional ejecutando un flujo determinista, sin
editar manualmente identificadores internos del workspace.

## Tabla de contenido

1. [Alcance y conceptos](#1-alcance-y-conceptos)
2. [Roles y responsabilidades](#2-roles-y-responsabilidades)
3. [Prerrequisitos](#3-prerrequisitos)
4. [Arquitectura del flujo dinámico](#4-arquitectura-del-flujo-dinámico)
5. [Convenciones de plantillas](#5-convenciones-de-plantillas)
6. [Procedimiento estándar de construcción](#6-procedimiento-estándar-de-construcción)
7. [Validación](#7-validación)
8. [Escalado a gemelos complejos](#8-escalado-a-gemelos-complejos)
9. [Solución de problemas](#9-solución-de-problemas)
10. [Rollback y limpieza](#10-rollback-y-limpieza)
11. [Checklist de referencia](#11-checklist-de-referencia)

---

## 1. Alcance y conceptos

Un gemelo digital en esta guía se compone de cuatro tipos de artefactos que se
tratan como **plantillas versionadas** en el repositorio:

| Artefacto | Ubicación | Rol en el gemelo |
|-----------|-----------|------------------|
| Modelos 3D | `3dmodels/` | Geometría de cada equipo (`.glb` / `.gltf`). |
| Scenes de TwinMaker | `scenes/` | Plantillas de escena con nodos, reglas y *data bindings*. |
| Dashboards de Grafana | `dashboards/` | Plantillas de visualización de telemetría y KPIs. |
| Alertas y datasources | `provisioning/` | Reglas de alerta y conexión a TwinMaker. |
| Mapeo de propiedades | `mapping/original_property_ids.csv` | Tabla puente entre nombres de negocio e IDs internos. |

**Software dinámico**: el script `scripts/migrate_files.py` es el motor que
resuelve en tiempo de despliegue los identificadores volátiles (entity IDs y
property IDs generados por TwinMaker/SiteWise) y el nombre del bucket, para que
las plantillas no contengan valores acoplados a una cuenta concreta.

## 2. Roles y responsabilidades

- **Autor de plantillas**: crea/edita scenes, dashboards y el CSV de mapeo siguiendo las [convenciones](#5-convenciones-de-plantillas).
- **Operador de despliegue**: ejecuta el [procedimiento estándar](#6-procedimiento-estándar-de-construcción) y la [validación](#7-validación).
- **Aprobador**: revisa el *diff* de plantillas y aprueba el merge/PR antes del despliegue en entornos compartidos.

## 3. Prerrequisitos

- Cuenta de AWS con acceso a IoT TwinMaker, IoT SiteWise, S3, IAM y EC2.
- Simulador SiteWise de la cervecería desplegado (ver `README.md`, paso 1.1), de modo que existan los modelos y activos de origen.
- Workspace de TwinMaker `BreweryManufacturing` creado (stack `cf/BreweryWorkspace.json`) y sincronizado con SiteWise.
- Python 3 con `boto3` y `awscli` configurados con credenciales/rol válidos.
- Permisos para `iottwinmaker:ExecuteQuery`, `iottwinmaker:CreateScene` y `s3:PutObject` sobre el bucket del workspace.

## 4. Arquitectura del flujo dinámico

El motor `migrate_files.py` ejecuta, en orden, los siguientes pasos
deterministas:

1. **Copia de recursos**: sube todos los modelos de `3dmodels/` al bucket del workspace (`copyResources`).
2. **Resolución dinámica de IDs**:
   - Lee el mapeo original desde `mapping/original_property_ids.csv` (`getOldPropertyIDs`).
   - Consulta el workspace con una *query* de EntityGraph para obtener los IDs **nuevos** por cada `entityName` + `displayName` (`getNewPropertyIDsFromTwinMaker`).
   - Reemplaza los IDs antiguos por los nuevos en scenes, dashboards y alertas (`replaceIDsInScenesAndDashboards`).
3. **Parametrización de bucket**: sustituye el marcador `{bucket}` por el bucket real en cada scene.
4. **Creación de scenes**: invoca `CreateScene` por cada archivo de `scenes/` y sube el contenido resultante a S3.

La clave es que las plantillas se escriben **una sola vez** con placeholders y el
software resuelve los valores concretos por entorno en el momento del despliegue.

## 5. Convenciones de plantillas

Para que el motor dinámico funcione, toda plantilla debe respetar estas reglas:

### 5.1 Scenes (`scenes/*.json`)

- El nombre del archivo (sin extensión) es el `sceneId` resultante; usar el nombre del equipo (p. ej. `MashTun.json`).
- Las referencias a modelos 3D deben usar el marcador de bucket: `"uri": "s3://{bucket}/<Modelo>.glb"`.
- Los *data bindings* parametrizan entidad y componente con `${sel_entity}` y `${sel_comp}`; no incrustar entity IDs literales fuera del `template` inicial.
- Las propiedades referenciadas deben usar los `propertyName` (`Property_<uuid>`) presentes en el CSV de mapeo, para que sean sustituibles.
- Las reglas (`rules`) se definen por expresión y color/icono, reutilizables entre equipos del mismo tipo.

### 5.2 Mapeo de propiedades (`mapping/original_property_ids.csv`)

- Cabecera obligatoria: `entityName,entityId,displayName,propertyName`.
- Cada fila es la clave `(entityName, displayName)` que el motor usa para emparejar el ID nuevo del workspace. Toda propiedad usada en scenes/dashboards/alertas debe tener su fila.

### 5.3 Dashboards y alertas (`dashboards/*.json`, `provisioning/alerting/alerts.yaml`)

- Referenciar propiedades por su `propertyName` del CSV para que la sustitución dinámica aplique.
- El datasource (`provisioning/datasources/twinmaker_datasource.yaml`) usa los marcadores `AWSREGION` e `IAMROLE`, resueltos por `install.sh`.

## 6. Procedimiento estándar de construcción

> Ejecutar desde `scripts/` en el servidor de despliegue.

1. Exportar la región y resolver el bucket del workspace:
   ```bash
   export AWS_DEFAULT_REGION=<region>
   export ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
   export BUCKET=twinmaker-workspace-brewery-$ACCOUNT_ID-$AWS_DEFAULT_REGION
   ```
2. (Opcional) Ejecutar `sh install.sh` para el flujo completo automatizado (modelos, IDs, rol IAM, datasource y Grafana). Para solo materializar plantillas:
   ```bash
   python3 migrate_files.py $BUCKET
   ```
3. El motor sube modelos, resuelve IDs, parametriza el bucket y crea las scenes en el workspace `BreweryManufacturing`.

> **Nota**: `migrate_files.py` modifica las plantillas *in situ* al sustituir IDs
> y `{bucket}`. Ejecutar siempre sobre una copia limpia del repositorio (checkout
> fresco) para no versionar valores específicos de una cuenta.

## 7. Validación

- **S3**: confirmar que los modelos de `3dmodels/` y los JSON de scenes existen en el bucket del workspace.
- **TwinMaker**: en la consola, verificar que cada scene aparece con su `sceneId` y renderiza la geometría sin IDs sin resolver.
- **IDs resueltos**: comprobar que ningún artefacto conserva `Property_<uuid>` antiguos ni el literal `{bucket}`.
- **Grafana**: abrir los dashboards y validar que llegan datos y que las alertas evalúan correctamente.

## 8. Escalado a gemelos complejos

Para añadir un equipo o línea nueva sin cambiar el motor:

1. Agregar el modelo 3D a `3dmodels/`.
2. Crear la scene plantilla en `scenes/` siguiendo las convenciones (placeholders de bucket, `${sel_entity}`/`${sel_comp}`, reglas reutilizables).
3. Añadir en `mapping/original_property_ids.csv` las filas de cada propiedad usada por la nueva scene/dashboard.
4. (Opcional) Añadir el dashboard en `dashboards/` y reglas en `provisioning/alerting/`.
5. Reejecutar el [procedimiento estándar](#6-procedimiento-estándar-de-construcción). El motor incorpora los nuevos artefactos automáticamente, ya que itera sobre los directorios.

Este patrón mantiene el software dinámico invariante mientras el gemelo crece en
número de activos y relaciones.

## 9. Solución de problemas

| Síntoma | Causa probable | Acción |
|---------|----------------|--------|
| Scene sin geometría | `{bucket}` no sustituido o modelo no subido | Reejecutar con `$BUCKET` correcto; verificar `copyResources`. |
| Propiedades vacías en scene/dashboard | Falta la fila en el CSV de mapeo | Añadir `(entityName, displayName)` y reejecutar. |
| `CreateScene` falla con "ya existe" | Scene creada en una corrida previa | Es idempotente para el contenido; el archivo igualmente se re-sube a S3. Borrar la scene en consola si se requiere recrearla. |
| IDs antiguos persisten | La *query* de EntityGraph no devolvió el par | Confirmar `componentName = 'sitewiseBase'` y que la sincronización SiteWise↔TwinMaker finalizó. |
| Grafana sin datos | Datasource con `AWSREGION`/`IAMROLE` sin resolver | Reejecutar `install.sh` o sustituir manualmente en el datasource. |

## 10. Rollback y limpieza

Seguir el orden del `README.md` (sección *Cleanup*):

1. Grafana Dashboard Server.
2. Rol IAM de Grafana.
3. Workspace de TwinMaker (borrar antes scenes y recursos en consola).
4. Servidor de simulación.
5. Activos de SiteWise.

Para revertir solo las plantillas, descartar los cambios *in situ* con
`git checkout -- scenes/ dashboards/ provisioning/` sobre el checkout usado.

## 11. Checklist de referencia

- [ ] Prerrequisitos cumplidos (workspace sincronizado, credenciales, `boto3`).
- [ ] Plantillas nuevas siguen las convenciones de la sección 5.
- [ ] Todas las propiedades usadas tienen fila en el CSV de mapeo.
- [ ] `migrate_files.py` ejecutado sobre un checkout limpio con `$BUCKET` correcto.
- [ ] Scenes creadas y modelos presentes en S3.
- [ ] Dashboards y alertas validados en Grafana.
- [ ] Plantillas sin IDs específicos de cuenta antes de confirmar en git.
