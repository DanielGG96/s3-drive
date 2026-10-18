# Especificación de Integración S3 para Hermes Desktop

## Resumen

Este documento describe las soluciones disponibles para conectar AWS S3 con Hermes Desktop, permitiendo operaciones de subida y descarga de archivos desde múltiples puntos de acceso: Hermes Desktop UI, terminal CLI, y potencialmente GUI/web.

---

## MCPS Disponibles

### 1. **ofershap/mcp-server-s3** ⭐ Recomendado
[Enlace al proyecto](https://claudemarketplaces.com/mcp/io.github.ofershap/s3)

#### Características
- Listar buckets y objetos
- Descargar/leer contenido de objetos como texto
- Subir contenido directamente (texto o archivos)
- Generar URLs firmadas presigned con caducidad personalizada
- Verificar existencia de buckets e información básica

#### Herramientas expuestas
| Herramienta | Descripción |
|-------------|-------------|
| `list_buckets` | Listar todos los buckets S3 en tu cuenta AWS |
| `list_objects` | Listar objetos en un bucket con filtro opcional por prefixo |
| `get_object` | Descargar y leer el contenido de un objeto como texto |
| `put_object` | Subir contenido de texto a un objeto S3 |
| `delete_object` | Eliminar un objeto del bucket |
| `presigned_url` | Generar URL temporal firmada para compartir objetos |
| `bucket_info` | Verificar si un bucket existe y obtener información básica |

#### Instalación en Hermes Desktop
```bash
npx mcp-server-s3
```

#### Configuración de credenciales
El servidor utiliza la cadena estándar S3:
1. Variables de entorno: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
2. Archivo compartido: `~/.aws/credentials` (en Windows: `C:\Users\daniel\.aws\credentials`)
3. Roles IAM: cuando se ejecuta en EC2, ECS, Lambda o similar

#### Permisos S3 requeridos
- `s3:ListBuckets`
- `s3:ListBucket`
- `s3:GetObject`
- `s3:PutObject`
- `s3:DeleteObject`
- `s3:HeadBucket`

---

### 2. **gangadharrr/aws-s3-mcp**
[Enlace al proyecto](https://github.com/gangadharrr/aws-s3-mcp)

#### Características
- Operaciones de objetos S3 básicas
- Listado, subida y descarga de objetos

#### Herramientas expuestas
| Herramienta | Descripción | Parámetros |
|-------------|-------------|------------|
| `list_objects` | Listar objetos en un bucket con filtro opcional de prefixo | `bucketName`, `prefix` |
| `upload_object` | Subir archivo o contenido a un bucket | `bucketName`, `key`, `filePath`, `content`, `contentType` |
| `download_object` | Descargar objeto desde el bucket | `bucketName`, `key`, `outputPath`, `returnContent` |

#### Configuración MCP (Claude Desktop / Hermes)
```json
{
  "mcpServers": {
    "aws-s3": {
      "command": "npx",
      "args": ["aws-s3"],
      "env": {
        "AWS_REGION": "us-east-1",
        "AWS_ACCESS_KEY_ID": "${YOUR_KEY}",
        "AWS_SECRET_ACCESS_KEY": "${YOUR_SECRET}"
      }
    }
  }
}
```

---

### 3. **dayongd1/mcp-s3** - File Manager completo
[GitHub repo](https://github.com/dayongd1/mcp-s3)

#### Características
- Operaciones avanzadas de gestión de archivos
- Seguimiento de progreso para archivos grandes
- Detección de conflictos y opción de sobrescritura forzada
- Soporte para URLs firmadas

#### Uso de comandos @mcp
```bash
@mcp Upload mi config.json archivo a S3
@mcp Subir backup.zip con expiración de 2 horas
@mcp Cargar report.pdf permitiendo sobrescribir forzosamente
@mcp Descargar config.json desde S3 a ./downloads/
@mcp Listar todos los archivos en S3
@mcp Mostrar info sobre config.json en S3
```

#### Ventajas
- Preserva nombres de archivo originales (`config.json` → `config.json`)
- Evita sobrescrituras accidentales (falla si el archivo existe)
- Opción de sobrescribir forzado para reemplazos intencionados
- Seguimiento de progreso para archivos grandes

---

## Implementación en Hermes Desktop

### Configuración Inicial

1. **Crear directorio MCP**:
   ```bash
   mkdir -p C:\Users\daniel\AppData\Local\hermes\profiles\default\mcpServers
   ```

2. **Elegir y configurar el MCP** (recomendado: `ofershap/mcp-server-s3`):
   ```json
   // Archivo: mcpServers/s3.json
   {
     "command": "npx",
     "args": ["mcp-server-s3"],
     "env": {
       "AWS_REGION": "us-east-1",
       "AWS_ACCESS_KEY_ID": "your-access-key-id",
       "AWS_SECRET_ACCESS_KEY": "your-secret-key"
     }
   }
   ```

3. **Instalación global (alternativa)**:
   ```bash
   pip install mcp-server-s3
   // Cambiar command en.json a: { "command": "python", "args": ["-m", "mcp-server-s3"] }
   ```

### Puntos de Acceso Habilitados

| Punto de acceso | Mecanismo | Descripción |
|-----------------|------------|-------------|
| **Hermes Desktop** | MCP integrado | Drag&drop entre paneles o panel de archivos S3 integrado |
| **Terminal CLI** | Herramienta hermes_tools | Comandos directos para listar/subir/bajar sin conexión MCP activa |
| **GUI/Web** | HTTP transport | Soporte si el backend expone endpoints REST/S3-compatible |

### Flujo de Trabajo Típico

#### Subir archivo a S3 desde Hermes Desktop
1. Usuario arrastra archivo al panel "S3 Upload" o usa comando natural
2. MCP `put_object` procesa la operación
3. Respuesta: `{"success": true, "bucket": "my-bucket", "key": "file.txt"}`

#### Descargar archivo desde S3
1. Usuario selecciona archivo en lista S3 y elige "Download"
2. MCP `get_object` descarga al directorio local especificado
3. Opción: devolver contenido como texto directamente (evita I/O de disco)

#### Listar archivos en bucket
1. Comando natural: "list files starting with logs/"
2. MCP `list_objects` con filtro por prefixo
3. Muestra paginación si hay más de 50 resultados

---

## Consideraciones de Seguridad

### Manejo de Credenciales
- **NUNCA** incluir credenciales reales en commit o repositorios compartidos
- Usar archivo `.env` local o variables de entorno del sistema
- En Hermes Desktop, almacenar en `profiles/<alias>/memories/.aws/credentials` con permisos 600

### Mejores Prácticas
- Principio de menor privilegio: usar IAM roles específicos (no root)
- Evitar storage lens publico; usar buckets privados con políticas apropiadas
- Generar URLs firmadas para compartir temporalmente en lugar de enlaces directos

---

## Referencias Externas

- [Open Source MCP Servers for AWS](https://awslabs.github.io/mcp/)
- [AWS - Build intelligent ETL pipelines with MCP and Amazon Q](https://aws.amazon.com/blogs/storage/build-intelligent-etl-pipelines-using-aws-model-context-protocol-and-amazon-q/)

---

## Estado de Implementación

- [x] Identificación de soluciones MCP para S3 completada
- [ ] Selección del MCP a implementar
- [ ] Instalación en Hermes Desktop (requiere credenciales AWS)
- [ ] Soporte terminal CLI disponible tras instalar MCP como herramienta
- [ ] Explorar soporte GUI/web depende de backend específico

## Próximos Pasos

1. **Obtener credenciales AWS** para la cuenta `daniel` (access key ID + secret access key)
2. **Elaborar decisión sobre qué MCP implementar** según requisitos específicos
3. **Configurar instalación inicial** en Hermes Desktop
4. **Pruebas operacionales** de subida, descarga y listado

---

*Documento generado: 2025-12-???*  
*Mantenido para integración S3 en Hermes Desktop*
