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
