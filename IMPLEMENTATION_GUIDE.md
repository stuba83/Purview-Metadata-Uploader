# Guía de Implementación

Esta guía detalla paso a paso cómo implementar la solución de actualización de metadatos de Purview desde Fabric.

## 📑 Tabla de Contenidos

1. [Prerequisitos](#prerequisitos)
2. [Configuración de Service Principal](#configuración-de-service-principal)
3. [Preparación de Datos](#preparación-de-datos)
4. [Configuración del Notebook](#configuración-del-notebook)
5. [Ejecución](#ejecución)
6. [Verificación](#verificación)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisitos

### Microsoft Fabric
- ✅ Workspace de Fabric con capacidad activa
- ✅ Lakehouse creado
- ✅ Permisos de Contributor en el workspace

### Microsoft Purview
- ✅ Cuenta de Purview aprovisionada
- ✅ Assets de Fabric ya escaneados en Purview
- ✅ Acceso administrativo a Purview

### Service Principal
- ✅ Azure AD App Registration
- ✅ Client ID y Client Secret generado
- ✅ Tenant ID

---

## Configuración de Service Principal

### 1. Crear App Registration en Azure AD

```bash
# Via Azure Portal
1. Azure Portal > Azure Active Directory > App registrations
2. Click "New registration"
3. Nombre: "purview-metadata-uploader"
4. Supported account types: Single tenant
5. Click "Register"
```

### 2. Generar Client Secret

```bash
1. En tu App Registration > Certificates & secrets
2. Click "New client secret"
3. Description: "Fabric-Purview Integration"
4. Expiration: 24 months (recomendado)
5. Click "Add"
6. ⚠️ COPIAR EL SECRET INMEDIATAMENTE (no se mostrará de nuevo)
```

### 3. Asignar Permisos en Purview

```bash
1. Purview Portal (https://web.purview.azure.com)
2. Data Map > Collections > Root collection
3. Role assignments
4. Add role assignment > Data Curator
5. Buscar tu Service Principal por nombre
6. Save
```

### 4. Asignar Permisos en Fabric

```bash
1. Fabric Portal > Tu Workspace
2. Workspace settings > Access
3. Add people or groups
4. Buscar tu Service Principal
5. Role: Contributor
6. Add
```

---

## Preparación de Datos

### Estructura de Tablas en Lakehouse

#### Tabla: `tablas_metadata`

```python
# Estructura requerida
{
    "Table": "string",              # Nombre de la tabla en Purview
    "Description": "string",        # Descripción del asset
    "Expert": "string",             # Email del experto
    "Owner": "string",              # Email del propietario
    "Source": "string",             # Sistema fuente
    "Refresh_Frequency": "string",  # Daily, Weekly, etc.
    "Data_Sensitivity": "string",   # High, Medium, Low
    "Retention_Period": "string",   # "3 years", "indefinite", etc.
    "Business_Owner": "string"      # Propietario de negocio
}
```

#### Ejemplo de Datos

```python
import pandas as pd

# Datos de ejemplo
data = {
    "Table": ["sales_transactions", "customer_master"],
    "Description": [
        "Transacciones de venta diarias", 
        "Maestro de clientes"
    ],
    "Expert": ["data-team@company.com", "data-team@company.com"],
    "Owner": ["sales-team@company.com", "crm-team@company.com"],
    "Source": ["SAP ERP", "Salesforce"],
    "Refresh_Frequency": ["Daily", "Hourly"],
    "Data_Sensitivity": ["High", "High"],
    "Retention_Period": ["7 years", "Indefinite"],
    "Business_Owner": ["Sales Department", "Marketing Department"]
}

df = pd.DataFrame(data)

# Guardar en Lakehouse
spark.createDataFrame(df).write.mode("overwrite").saveAsTable("tablas_metadata")
```

---

## Configuración del Notebook

### 1. Importar Notebook a Fabric

```bash
1. Fabric Portal > Tu Workspace
2. New > Import notebook
3. Seleccionar PurviewMetadataUploader.ipynb
4. Upload
```

### 2. Adjuntar Lakehouse

```bash
1. Abrir el notebook
2. Click en "Add" en el panel izquierdo
3. Seleccionar tu Lakehouse
4. Add
```

### 3. Configurar Variables

Edita la **Primera Celda** del notebook:

```python
# ===== CONFIGURACIÓN =====
PURVIEW_ACCOUNT_NAME = "your-purview-account"
TENANT_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
CLIENT_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
CLIENT_SECRET = "your-secret-value"
YOUR_EMAIL = "admin@yourcompany.com"
BUSINESS_METADATA_NAME = "DataGovernance"
```

⚠️ **IMPORTANTE**: 
- En producción, usa Azure Key Vault
- Nunca hagas commit de estos valores

---

## Ejecución

### Proceso Paso a Paso

#### 1. Verificar Conexión

Ejecuta las primeras celdas para verificar:
- ✅ Autenticación con Purview
- ✅ Lectura de datos del Lakehouse
- ✅ Funciones cargadas correctamente

#### 2. Crear Business Metadata Definition

El notebook automáticamente:
1. Verifica si "DataGovernance" existe
2. Si no existe, lo crea
3. Asocia los atributos al tipo `fabric_lakehouse_table`

```python
# Output esperado
================================
CREANDO BUSINESS METADATA
================================
Status: 200
✅ Business Metadata creado CORRECTAMENTE
```

#### 3. Test con 1 Tabla

Antes de procesar todo, prueba con una tabla:

```python
# Output esperado
================================
TEST CON 1 TABLA
================================
[1/1] sales_transactions
  ✅ OK
```

#### 4. Procesar Todas las Tablas

Si el test es exitoso, ejecuta la celda final:

```python
# Output esperado
================================
PROCESANDO 5 TABLAS
================================
[1/5] sales_transactions
  ✅ OK
[2/5] customer_master
  ✅ OK
...

================================
RESUMEN FINAL
================================
Total tablas: 5
Actualizadas: 5
Errores: 0
```

---

## Verificación

### En Purview Portal

1. **Abrir Purview**
   ```
   https://web.purview.azure.com
   ```

2. **Buscar Asset**
   - Search bar > Nombre de tu tabla
   - Click en el resultado

3. **Verificar Metadata**
   - Tab: **Overview**
   - Scroll down: **Data asset attributes**
   - Expandir: **DataGovernance**
   - Verificar todos los campos

### Campos Esperados

- ✅ Expert
- ✅ Owner
- ✅ Source
- ✅ Refresh_Frequency
- ✅ Data_Sensitivity
- ✅ Retention_Period
- ✅ Business_Owner

---

## Troubleshooting

### Error: "Authentication failed"

**Causa**: Service Principal sin permisos

**Solución**:
```bash
1. Verificar rol "Data Curator" en Purview
2. Verificar "Contributor" en Fabric Workspace
3. Esperar 5-10 minutos para propagación de permisos
```

### Error: "Table not found in Purview"

**Causa**: Asset no está en el catálogo

**Solución**:
```bash
1. Purview Portal > Data Map
2. Sources > Tu Fabric workspace
3. Verificar que el scan está completo
4. Si no existe, ejecutar nuevo scan
```

### Error: "Invalid business-metadata for entity type"

**Causa**: Business Metadata no asociado al tipo correcto

**Solución**:
```bash
1. Verificar que usaste applicableEntityTypes en la definición
2. El valor debe ser: '["fabric_lakehouse_table"]'
3. Recrear el Business Metadata si es necesario
```

### Error: "No business attributes visible in UI"

**Causa**: Caché de UI

**Solución**:
```bash
1. Hard refresh: Ctrl + Shift + R
2. Cerrar y reabrir el tab
3. Esperar 5 minutos para propagación
```

---

## Mejores Prácticas

### 1. Seguridad

- ✅ Usar Azure Key Vault para secrets
- ✅ Rotar Service Principal secrets cada 6 meses
- ✅ Aplicar principio de least privilege
- ✅ Auditar accesos regularmente

### 2. Datos

- ✅ Validar emails antes de upload
- ✅ Usar valores estandarizados (High/Medium/Low)
- ✅ Mantener datos de metadata actualizados
- ✅ Documentar cambios en descripción

### 3. Operación

- ✅ Ejecutar en horarios de bajo tráfico
- ✅ Monitorear logs de ejecución
- ✅ Mantener backup de metadata
- ✅ Probar cambios en dev antes de prod

---

## Próximos Pasos

1. **Metadata de Columnas**: Extender para actualizar metadata a nivel de columna
2. **Glossary Terms**: Vincular términos del glosario de negocio
3. **Classifications**: Asignar clasificaciones automáticas
4. **Lineage**: Enriquecer con información de lineage

---

## Recursos Adicionales

- [Microsoft Purview REST API](https://learn.microsoft.com/en-us/rest/api/purview/)
- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)
- [Business Metadata Guide](https://learn.microsoft.com/en-us/azure/purview/how-to-business-glossary-custom-attributes)
