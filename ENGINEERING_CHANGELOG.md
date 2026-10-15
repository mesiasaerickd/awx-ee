# Registro de ingeniería y seguridad — AWX Execution Environment

Fecha de apertura: 2026-10-13
Estado: diagnóstico inicial; **sin cambios de código de aplicación ni imágenes construidas**.

## Política de ramas
- `devel`: referencia heredada del proyecto upstream, no modificar.
- `dev`: análisis y cambios sujetos a pruebas.
- `prd`: solo código validado, por ahora idéntico al punto inicial de `devel`.

## Hallazgos iniciales
Se verificó `execution-environment.yml`: base CentOS Stream 9 y Python 3.12, Ansible Core `<2.19`, colecciones Galaxy y dependencias de ejecución. Usa referencias de desarrollo sin fijar para algunos paquetes, que requieren pinning para compilaciones reproducibles.

## Decisiones
1. Comparar el código `devel` con la imagen desplegada AWX 24.6.1 antes de cambiar imagen base.
2. No cambiar imágenes de Kubernetes productivo ni tocar el servidor actual.
3. Mantener cambios en PR desde `dev` hacia `prd`, con revisión, SBOM, análisis Trivy, pruebas de construcción y pruebas funcionales.
4. Construir imágenes en CI/runner aislado y publicarlas en un registro OCI, identificadas por SHA256 y versión propia. Ninguna imagen queda aprobada sin pruebas.
5. Evaluar CentOS Stream 9 y UBI 9 por separado: sustituir el sistema base no corrige por sí mismo dependencias Python.

## Historial
- 2026-10-13: creadas ramas `dev` y `prd` desde `devel`; inspección inicial de fuentes y apertura del registro de ingeniería.

## Pendiente
- Verificar árbol de dependencias resueltas, compilar, obtener SBOM y ejecutar Trivy.
- Establecer runners y condiciones de publicación de imágenes.
- Verificar tests, compatibilidad AWX/Operator/EE y recuperación de PostgreSQL.
- Decidir controles y riesgos aceptables antes de solicitar despliegue en producción.

- 2026-10-13: se agregó el workflow de GitHub Actions para compilación de imagen en `dev`, sin despliegue ni publicación al registro. Estado: ejecución y validación pendientes; fallo de build no autoriza cambios automáticos a `prd`.

- 2026-10-13: se extendió GitHub Actions para escanear la imagen compilada con Trivy 0.75.0 (CRITICAL y HIGH), generar un JSON y subirlo como artifact durante 30 días. Los hallazgos no bloquean todavía el build: se requiere establecer baseline, validar límites de cobertura y definir umbrales antes de habilitar gates. Sin push de imágenes ni modificaciones en `prd`.

## Verificación de CI y seguridad — 2026-10-13
Run 38493349345: build y Trivy completados. CRITICAL=0, HIGH=23. Artifact `awx-ee-trivy-report`, ID 11862647899. Trivy advierte sobre SBOM de terceros y falta de cobertura OS CentOS Stream 9.

Decisión: mantener `prd` sin cambios; no publicar a GHCR hasta identificar CVE remanentes, comprobar cobertura OS y validar la integración Kubernetes. La ejecución de Trivy exitosa significa que el escáner pudo correr, no que no existan vulnerabilidades.

## Mitigación Python EE y trazabilidad Receptor — 2026-10-14
- Se extrajo y examinó el JSON original del artifact `awx-ee-trivy-report` (ID 11862647899; run #38493349345). Base: 0 CRITICAL / 23 HIGH: **7 HIGH Python** y **16 HIGH Receptor (Go)**. El escáner no proporcionó cobertura fiable de vulnerabilidades del SO CentOS Stream 9.
- Hallazgos Python en reporte: `cryptography 47.0.0` (3 HIGH; fix >=50.0.0 para la más reciente); `msgpack 1.1.2` (1 HIGH; fix 1.2.1); `setuptools 70.3.0` (1 HIGH; fix 78.1.1); `urllib3 2.7.0` (2 HIGH; fix 2.8.0).
- `execution-environment.yml` ahora actualiza las distribuciones en imagen final a `cryptography==50.0.0`, `msgpack==1.2.1`, `setuptools>=78.1.1`, `urllib3==2.8.0` y ejecuta `pip check`; versiones y dependencias se validan en el workflow. Candidato pendiente de prueba integral; no declararlo reparado hasta nuevo Trivy.
- Receptor: 16 HIGH embebidas en `/usr/bin/receptor`: `golang.org/x/crypto v0.51.0` (10), `x/net v0.55.0` (2), `x/text v0.37.0` (1) y Go stdlib v1.26.7 (3). Trivy refiere correcciones hasta `x/crypto v0.55.0`, `x/net v0.60.0`, `x/text v0.39.0` y Go 1.26.9. Se añade extracción de `go version -m` en CI para identificar módulos exactos antes de reconstruir el binario.
- Cambios solo en `dev`. CI construye en local sin publicar imagen ni modificar `prd`, `devel`, GHCR o Kubernetes.

## Resolución de conflicto cryptography/MSAL/pyOpenSSL — 2026-10-14
- Los runs #38652531639, #38652548050 y #38652568186 terminaron con fallo en `Build execution environment without publishing`. El log de #38652568186 confirma que `msal 1.33.0b1` requiere `cryptography<48` y `pyopenssl 26.2.0` requiere `cryptography<49`, incompatibles con la actualización propuesta a `cryptography 50.0.0`. La compilación se detuvo por `pip check`, evitando publicar una imagen con dependencias rotas.
- Se revisaron versiones publicadas en PyPI y la documentación de pyOpenSSL 26.4.0, que admite `cryptography 50.x`. Se añadió `msal==1.39.0` y `pyOpenSSL==26.4.0` a la instalación final junto a los parches Python. La compatibilidad real de MSAL debe ser demostrada por `pip check` en CI, no asumida.
- El cambio está solo en dev y requiere ejecución verde del build, extracción Receptor y nuevo Trivy antes de afirmar disminución de HIGH. Sin cambios en prd, devel, GHCR ni Kubernetes.

## Conflicto Azure CLI y reversión de cryptography — 2026-10-14
- Run #38654287609 FAILED: `azure-cli-core 2.75.0` requiere exactamente `msal==1.33.0b1`, incompatible con el intento `msal==1.39.0`; esta versión anterior de MSAL restringe cryptography, por lo que no se puede forzar cryptography 50 sin rehacer la cadena Azure CLI.
- Se conservan `msal` y `cryptography` originales del EE; se mantiene actualización de `msgpack==1.2.1`, `setuptools>=78.1.1`, `urllib3==2.8.0` y `pip check`, y se retiró la aserción incompatible de cryptography 50.
- Compromiso explícito: la corrección de los 3 HIGH en cryptography se pospone hasta tratar Azure CLI/MSAL/pyOpenSSL de forma compatible. Esta revisión se debe validar con build y Trivy; no afirmar reducción aún.
- Solo dev; nada publicado a GHCR ni desplegado a Kubernetes.

## Build EE restaurado; Trivy no disminuye — 2026-10-14
- Ejecución #38655445281: SUCCESS, build de imagen y `pip check` validados. Artifact `awx-ee-trivy-report` 11937036756.
- Logs: Receptor sigue con `go1.26.7 (Red Hat 1.26.7-2.el9_8)`; Trivy `CRITICAL=0`, `HIGH=23`. No afirmar que las 7 HIGH Python se mitigaron basándose solo en un build verde; revisar contenido JSON de la nueva imagen y versiones/ubicaciones efectivas de paquetes antes de otro cambio.
- Siguiente bloque: reconstrucción de Receptor Go a partir de código y dependencias compatibles, sin publicación a GHCR ni cambios en prd o Kubernetes.

## Candidato de mitigación Receptor Go — 2026-10-14
- Se creó `Dockerfile.receptor`: reconstruye `/out/receptor` desde `ansible/receptor` rama devel en commit anclado `8a95593179b27454b7e46629e91c59d9a73a9620`, con compilador `golang:1.26.9`, módulos `golang.org/x/crypto@v0.55.0`, `x/net@v0.60.0`, `x/text@v0.42.0`, `go mod tidy` y `CGO_ENABLED=0`.
- `execution-environment.yml` reemplaza el COPY del ejecutable Receptor heredado de `quay.io/ansible/receptor:devel` por un COPY de la imagen temporal local `awx-ee-receptor-patched:ci`. Se mantienen el resto de dependencias y correcciones Python existentes.
- El workflow `.github/workflows/ee-build.yml` compila esa imagen temporal primero, comprueba versión del ejecutable y posteriormente construye EE, ejecuta pip check, inventario Go y Trivy. No publica la imagen temporal ni EE a un registro.
- Candidato aún SIN validar: no afirmar remediación de las 16 HIGH hasta completar build y revisar Trivy. El reporte anterior sigue 0 CRITICAL / 23 HIGH, con cobertura limitada para CentOS Stream 9.
- Exclusivamente dev; sin modificaciones en prd, devel, GHCR o Kubernetes.

## Corrección de dependencia Go en Receptor — 2026-10-14
- Run #38656843517 falló al no existir todavía la imagen local temporal que utiliza el COPY; el workflow fue corregido para compilar primero `awx-ee-receptor-patched:ci`.
- Run #38656880686 falló en build de Receptor porque `golang.org/x/net@v0.60.0` requiere `golang.org/x/crypto@v0.57.0`, incompatible con el pin anterior 0.55.0. Se actualiza el pin a `x/crypto@v0.57.0` en `Dockerfile.receptor`.
- Son correcciones de build exclusivamente en dev, no resultados de seguridad. Pendiente un nuevo build completo, controles Python y Trivy para confirmar reducción de CVE.

## Validación exitosa de Receptor reconstruido — 2026-10-14
- GitHub Actions #38656969887: build AWX EE SUCCESS, validación Python y captura de Receptor Go SUCCESS, Trivy ejecutado y artefacto `awx-ee-trivy-report` ID 11938115432.
- Los logs confirman Receptor con Go 1.26.9 y módulos `golang.org/x/crypto 0.57.0`, `x/net 0.60.0`, `x/text 0.42.0`.
- Trivy confirma CRITICAL=0 y HIGH=7, reducidos desde 23 HIGH; 16 hallazgos menos tras reconstruir Receptor. La cobertura del SO CentOS Stream 9 sigue siendo limitada. Las 7 HIGH deben investigarse con el JSON nuevo, sin asumir que todas son cryptography.
- La imagen fue compilada únicamente en CI sin publicación; no se ha validado el despliegue en Kubernetes ni promovido a prd.

## Investigación de las 7 HIGH Python residuales — 2026-10-14
- Se leyó el JSON real del artefacto 11938115432, run #38656969887. Trivy identifica exclusivamente 7 HIGH en `Python (python-pkg)`: cryptography 47.0.0 (3), msgpack 1.1.2 (1), setuptools 70.3.0 (1), urllib3 2.7.0 (2). No son hallazgos Go del Receptor.
- El build había aprobado comprobaciones Python de versiones más nuevas, lo que sugiere instalaciones o metadatos duplicados; **esa hipótesis aún debe comprobarse**. No se deben suprimir entradas del reporte ni borrar metadatos antes de determinar cuál versión se importa o es utilizada por la imagen.
- Se agregó al workflow de dev una etapa de diagnóstico de distribuciones Python (`importlib.metadata`, `sys.path` y búsqueda de `dist-info` en la imagen final) antes de Trivy. Tras examinar la salida se decidirá corrección de duplicados sin ocultar vulnerabilidades.
- No se ha publicado imagen ni promovido a prd o Kubernetes.

## Próximo ensayo Azure CLI y hallazgos SBOM — 2026-10-14
- Se examinó el JSON real de Trivy, artifact 11938880097 del run #38658375860. Hallazgos HIGH: 3 en cryptography 47.0.0 con ruta real `usr/local/lib64/python3.12/site-packages/...`; 1 en msgpack 1.1.2, 1 en setuptools 70.3.0 y 2 en urllib3 2.7.0, estos cuatro etiquetados `AnalyzedBy=sbom` sin FilePath, pese a que el mismo escaneo muestra metadatos actuales de msgpack 1.2.1, setuptools 84.0.0 y urllib3 2.8.0. Se conservarán los hallazgos y no se excluirá ni borrará SBOM para falsear el resultado.
- Candidato experimental: resolver conjuntamente `azure-cli>=2.75.0`, `pyOpenSSL>=26.4.0` y `cryptography==50.0.0` mediante `pip install --upgrade`, manteniendo los pins corregidos de los otros paquetes. Se exige `pip check` y el workflow registra las versiones de Azure CLI, MSAL y pyOpenSSL y comprueba cryptography 50. No se ha verificado todavía la compatibilidad efectiva; el build debe fallar si el resolvedor no la consigue.
- Este cambio es exclusivo de dev, sin imagen publicada o cambios en prd/Kubernetes. Tras CI exitoso se requieren pruebas funcionales de Azure y de la Execution Environment antes de promoción.

## Validación Azure CLI y cryptography — 2026-10-14
- Workflow #38661640290 SUCCESS: build EE, control `pip check`, comprobaciones de paquetes Python, Receptor Go 1.26.9 y Trivy completos. Artifact `awx-ee-trivy-report` ID 11939447819.
- Runtime confirmado: azure-cli 2.91.0, msal 1.39.0, pyOpenSSL 26.4.0, cryptography 50.0.0, msgpack 1.2.1, setuptools 84.0.0, urllib3 2.8.0.
- Trivy: CRITICAL=0 y HIGH=4, desde baseline 23 HIGH. Los cuatro hallazgos residuales todavía requieren clasificación mediante el JSON; no silenciarlos ni declararlos corregidos sin evidencia.
- La imagen solo se construyó en CI, no se publicó ni probó en Kubernetes; dev exclusivamente.

## Cierre parcial residual SBOM y prevención de regresiones — 2026-10-14
- Se analizó el JSON original Trivy del run #38661640290, artifact 11939447819. Las cuatro HIGH son GHSA-6v7p-g79w-8964 (msgpack 1.1.2), CVE-2025-47273 (setuptools 70.3.0), CVE-2026-97687 y CVE-2026-97689 (urllib3 2.7.0). Estos elementos figuran en `Results[].Packages` con `AnalyzedBy=sbom`, sin `FilePath`, todos en la capa `sha256:c077b26c7a36898cb8786665578d2c08674344de0f378ef7545ee0ef3194a765`. En el mismo informe hay entradas `AnalyzedBy=python-pkg` de versiones actualizadas en rutas reales: msgpack 1.2.1, setuptools 84.0.0, urllib3 2.8.0. No se interpreta esto como prueba de ausencia total de código vulnerable en cada capa, y las cuatro HIGH se mantienen contadas.
- Se añaden smoke tests de `ansible --version`, `ansible-runner --version`, `az version`, `receptor --version` y `pip check` sobre imagen construida, además de gate de regresión Trivy: fallo cuando CRITICAL > 0 o HIGH > 4. No se excluyen findings ni se manipula SBOM. Gate de baseline no implica cero vulnerabilidades.
- Requiere CI verde antes de promover nada; faltan además pruebas de integración real en Kubernetes y cobertura del SO CentOS Stream 9.

## Ajuste del smoke test de Ansible — 2026-10-14
- Run #38663680118 falló únicamente en `Smoke-test execution environment commands`: `ansible --version` intentó crear `/.ansible/tmp` y recibió Permission denied. La prueba se lanzaba con HOME no escribible en el contenedor; no se confirmó fallo del build ni incremento de CVEs.
- Se establece `HOME=/tmp` y `ANSIBLE_LOCAL_TEMP=/tmp/ansible-local` en el smoke test y se crea su carpeta antes de invocar Ansible. Sin alterar permisos del runtime productivo. Se requiere repetir workflow completo.

## Revalidación de smoke tests — 2026-10-14
- GitHub Actions #38664721890 SUCCESS, build y smoke tests sin pasos fallidos. Ansible core 2.18.19; dependencias `pip check` sin conflictos; Azure CLI 2.91.0, MSAL 1.39.0, pyOpenSSL 26.4.0; Receptor Go 1.26.9.
- Trivy: CRITICAL=0, HIGH=4; gate de regresión aprobado. Artifact `awx-ee-trivy-report` ID 11940444544.
- La verificación es de CI y comandos básicos, **no** prueba funcional Kubernetes ni certificación de ausencia de vulnerabilidades. Las cuatro HIGH SBOM persisten. Sin promoción a prd, publicación de imágenes ni despliegue.

## Ensayo de mitigación de 4 hallazgos SBOM residuales — 2026-10-14
- Se inspeccionó el JSON de Trivy del artifact #11939447819. Los cuatro hallazgos remanentes pertenecen a `AnalyzedBy=sbom`, todos asociados a la misma capa antigua `sha256:c077b26c7a36898cb8786665578d2c08674344de0f378ef7545ee0ef3194a765`: msgpack 1.1.2 (GHSA-6v7p-g79w-8964), setuptools 70.3.0 (CVE-2025-47273) y urllib3 2.7.0 (CVE-2026-97687, CVE-2026-97689). Las comprobaciones del runtime muestran las versiones corregidas, pero los cuatro siguen siendo hallazgos de Trivy y no se excluyen.
- Ensayo en dev: añadir `msgpack==1.2.1`, `setuptools>=78.1.1` y `urllib3==2.8.0` directamente a `dependencies.python` en `execution-environment.yml`, para intentar que se resuelvan antes de que Ansible Builder componga las capas y SBOM. Se mantienen comprobaciones finales y escaneo sin exclusiones.
- No está validado que la modificación elimine el registro heredado: se requiere build exitoso, smoke tests y nuevo Trivy para determinarlo. No se ha tocado prd, GHCR ni Kubernetes.

## Resultado de mitigación de dependencias iniciales — 2026-10-14
- Run #38667041096 SUCCESS: compilación AWX EE, pruebas básicas y `pip check` aprobados. Receptor Go 1.26.9; Azure CLI 2.91.0; MSAL 1.39.0; pyOpenSSL 26.4.0. Artifact Trivy 11942102523.
- Trivy mantiene CRITICAL=0 / HIGH=4 después de introducir versiones corregidas de msgpack, setuptools y urllib3 en `dependencies.python`; el intento no redujo el conteo. No deben clasificarse los cuatro como corregidos ni suprimirse.
- Siguiente investigación: identificar qué dependencia/imagen o contenido SBOM introduce la capa antigua y evaluar corrección en origen, sin ocultar resultados. Continúa pendiente integración Kubernetes. Solo dev, no publicar ni desplegar.

## Rastreo de la capa SBOM heredada — 2026-10-14
- Se añadió una etapa CI en `dev` que registra las capas RootFS (`docker image inspect`), historial de construcción (`docker history --no-trunc`), etiquetas OCI y posibles ficheros SBOM. Estos datos se guardan junto al JSON Trivy en el artefacto de evidencia.
- Objetivo: atribuir los 4 HIGH asociados por Trivy a la capa sha256:c077b26c7a36898cb8786665578d2c08674344de0f378ef7545ee0ef3194a765; no eliminar paquetes, ocultar hallazgos, modificar controles de regresión ni aceptar supuestamente 0 HIGH sin evidencia.
- Etapa de diagnóstico pendiente de validar. No se cambian `prd`, GHCR ni Kubernetes.

## Trazabilidad de etapas Ansible Builder — 2026-10-14
- Se amplió el CI `dev` para guardar `context/Dockerfile` generado por Ansible Builder, relación de etapas relevantes (FROM, RUN, COPY, pip y sbom) y salida íntegra `docker history --no-trunc` sin recortes.
- El objetivo es atribuir los 4 HIGH que Trivy asocia a SBOM heredado a una etapa concreta antes de modificar fuentes, bases o metadatos. Se mantiene Trivy íntegro, sin exclusiones ni reducción artificial de conteos.
- Pendiente comprobar la ejecución y extraer conclusiones a partir de los artefactos. Sin modificaciones de imágenes de producción, prd, GHCR ni Kubernetes.

## Verificación de evidencia Ansible Builder — 2026-10-14
- Run #38669897567 SUCCESS, pruebas básicas y `pip check` aprobadas; versiones verificadas Azure CLI 2.91.0, MSAL 1.39.0, pyOpenSSL 26.4.0 y Receptor Go 1.26.9.
- `context/Dockerfile` capturado exitosamente (3361 bytes), historial de imagen, capas y Trivy disponibles en artifact 11943827288. Trivy informa CRITICAL=0 y HIGH=4, sin cambios.
- La atribución concreta de los cuatro metadatos SBOM a una instrucción o stage todavía necesita inspección detallada del Dockerfile/artefacto. No declarar mitigación ni cero hallazgos por esta captura. Solo dev; sin cambios productivos.

## Inspección forense del SBOM embebido — 2026-10-14
- Inspección del ZIP real GitHub Actions artifact 11943827288, `trivy-reports/awx-ee-trivy.json`: las cuatro HIGH están marcadas `AnalyzedBy=sbom`, sin FilePath, en DiffID `sha256:49b04df1247d051f6fa49b2e83169fb066411bbe88c6db8087139d59da0f81b3`: msgpack 1.1.2, setuptools 70.3.0, urllib3 2.7.0. Las versiones activas corregidas se detectan por `python-pkg` en otra capa. Los antiguos paquetes no están identificados como distribuciones Python instaladas en la capa final.
- El mismo artefacto identifica SBOM embebidos en `cryptography-50.0.0.dist-info/sboms/`, entre otros. Se añadió un paso de diagnóstico que examina de manera de solo lectura los SBOM JSON embebidos y lista coincidencias de PURL para los tres paquetes. Esta correspondencia es una hipótesis pendiente de probar en CI, no conclusión definitiva.
- No se borraron ni filtraron SBOM; se mantiene Trivy íntegro y gate de no regresión. Sin cambios en producción, imágenes publicadas ni prd.

## Resultado de inspección SBOM embebido — 2026-10-14
- Run #38671764750 SUCCESS; build, pruebas básicas, comprobación de dependencias y Trivy completados. Artifact `awx-ee-trivy-report` ID 11945275830; CRITICAL=0/HIGH=4.
- El paso `Inspect embedded package SBOM provenance` buscó referencias a msgpack, setuptools y urllib3 en SBOM JSON de cryptography 50.0.0, pydantic_core 2.50.1 y rpds_py 2026.9.1: no encontró coincidencias ni PURL relevantes. La hipótesis de que estos SBOM concretos causaban las cuatro detecciones no se confirmó; no eliminar dichos documentos ni alterar el escaneo.
- Pendiente atribuir las cuatro entradas `AnalyzedBy=sbom` al origen exacto en capas/metadata y decidir remediación verificable. Solo rama dev; sin despliegue/publicación.

## Atribución detallada de hallazgos Trivy SBOM — 2026-10-14
- Añadida etapa `Trace each Trivy finding to SBOM package source` en el workflow `dev`. Extrae del JSON original de Trivy por cada CVE: target, clase, tipo, PkgID, paquete/versión, layer e información `AnalyzedBy`, `FilePath` y `Identifier` de `Results[].Packages` coincidentes.
- Evidencia preservada como `security-evidence/trivy-sbom-attribution.json` sin modificar el reporte de Trivy, sus umbrales ni los hallazgos originales. La siguiente ejecución permitirá distinguir de manera auditable metadatos SBOM versus archivos Python activos.
- No se declara remediación de las 4 HIGH hasta contar con prueba técnica verificable; no se publican imágenes ni se cambian `prd`, GHCR ni Kubernetes.

## Verificación de atribución individual SBOM — 2026-10-14
- GitHub Actions #38673377469 SUCCESS, reporte Trivy CRITICAL=0/HIGH=4, artifact 11944884131. La etapa de atribución ejecutó correctamente y registró PkgID, PURL, AnalyzedBy, FilePath y DiffID por hallazgo.
- Los cuatro comparten en **esa compilación** DiffID `sha256:f3276c9fd4de240f6f63f9041b3098ee7456388ea659895bd91094a928ff12bf`, `AnalyzedBy=sbom` y `FilePath=null`: msgpack@1.1.2 GHSA-6v7p-g79w-8964; setuptools@70.3.0 CVE-2025-47273; urllib3@2.7.0 CVE-2026-97687 y CVE-2026-97689.
- El mismo DiffID en un build concreto no demuestra falso positivo ni identifica aún la instrucción que generó los registros. Próxima prueba: mapear la capa del build con el Dockerfile generado e identificar el originador SBOM antes de cualquier remoción o modificación. No se filtran hallazgos.
- Solo dev; sin publicación, despliegue ni modificación de prd.

## Ensayo de corrección en etapa base de Ansible Builder — 2026-10-14
- Se abrió y analizó el artifact completo del run #38673377469 (11944884131), incluyendo JSON de Trivy, `image-layers.json`, Dockerfile generado e historial Docker. Los 4 HIGH del SBOM comparten DiffID `sha256:f3276c9fd4de240f6f63f9041b3098ee7456388ea659895bd91094a928ff12bf`, índice 7 de 22 capas RootFS (índice base cero). Esa capa es previa a las actualizaciones finales, pero el índice por sí solo no identifica de forma incontrovertible una instrucción Dockerfile.
- Se añadió a `additional_build_steps.append_base` la actualización de `msgpack==1.2.1`, `setuptools>=78.1.1` y `urllib3==2.8.0`, más `pip check`. Se mantienen los pins en la resolución de dependencias y las verificaciones de la imagen final.
- La intervención intenta prevenir componentes antiguos en etapas heredadas de Builder, no suprimir o ignorar CVE. Debe validarse build, controles funcionales y Trivy; si la imagen base conserva metadatos históricos, todavía puede haber 4 HIGH.
- Cambios solo en dev; no modificar prd, GHCR ni Kubernetes.

## Resultado del ensayo append_base — 2026-10-14
- GitHub Actions #38675419951 SUCCESS: compilación, dependencias Python y controles del workflow aprobados. Runtime confirma Azure CLI 2.91.0, MSAL 1.39.0, pyOpenSSL 26.4.0, Receptor Go 1.26.9; `pip check` sin problemas.
- Trivy reporta CRITICAL=0/HIGH=4 pese a actualizar paquetes en `append_base`. Artifact de evidencia 11946637400. Este intento no disminuyó hallazgos; queda pendiente investigación de la imagen/capa de origen y cobertura de las detecciones SBOM.
- No se considera cero vulnerabilidades ni listo para producción. Cambios solo en dev; sin publicar ni desplegar.

## Ensayo de reconstrucción con imagen base alternativa — 2026-10-14
- Tras sucesivos ensayos de actualización de los tres paquetes Python sin reducción de 4 HIGH SBOM heredadas, se cambia en `dev` solamente la base de Ansible Builder de `quay.io/centos/centos:stream9` a `registry.access.redhat.com/ubi9/ubi:9.8`, imagen de base con soporte de proveedor. El propósito es reconstruir capas de origen y medir objetivamente si desaparecen los cuatro hallazgos.
- Este cambio **puede fallar** por disponibilidad de repositorios/EPEL o introducir nuevas CVE de sistema RPM. Se mantiene el escaneo Trivy completo (sin exclusiones) y el gate de 0 CRITICAL y máximo 4 HIGH, además del smoke test. No se aceptará un resultado limpio sin confirmar también cobertura del SO.
- Candidato estrictamente experimental: ninguna imagen se publica ni se despliega. Se revertirá en dev si los controles fallan o introduce regresiones.
