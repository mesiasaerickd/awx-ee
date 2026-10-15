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
