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
