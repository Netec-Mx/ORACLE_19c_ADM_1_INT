# Prática 1.4. SQL-Developer (opcional)

<br/><br/>

## Objetivos
Al finalizar la práctica, serás capaz de:
- Verificar que SQL Developer se encuentre instalado.
- Instalar SQL Developer en caso de que no se encuentre instalado.

<br/><br/>

## Tiempo estimado
- 120 minutos.

<br/><br/>

## Objetivo visual 

![diagrama1](../images/img21.png)


<br/><br/>

## Tabla de ayuda

### **Descarga de SQL Developer**

| **Software**             | **Descripción**                                                                | **Enlace de descarga**                                                                        |
| ------------------------ | ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| **Oracle SQL Developer** | Herramienta gráfica para desarrollo y administración de bases de datos Oracle. | [Descargar desde Oracle](https://www.oracle.com/database/sqldeveloper/technologies/download/) |

<br/><br/>

## Instrucciones 

#### **Paso 1.** Asegúrate de tener instalado un **JDK**. Si no lo tienes, descárgalo (SQL Developer soporta **Oracle JDK 17**).
   ```bash
   su -
   password: Oracle1
   cd Downloads
   # Verifica que tengas instalado el JDK de Java 17
   java --version

   # En caso de que no exista o tengas una version anterior a la 17 ejecuta el siguiente comando
   rpm -i  jdk-17.0.10_linux-x64_bin.rpm
   ```
#### **Paso 2.** Instala el paquete de SqlDeveloper ejecutando el comando:

   ```bash
   rpm -Uhv sqldeveloper-(build number)-1.noarch.rpm
   ```

#### **Paso 3.** Accede al directorio de instalación.

   ```bash
   whereis sqldeveloper
   ```

#### **Paso 4.** Ejecuta el script de inicio.

   ```bash
   /usr/local/bin/sqldeveloper
   ```

#### **Paso 5.(Opcional)** Cuando se te solicite, introduce la ruta del `JDK`, por ejemplo:

   ```bash
   /usr/java/jdk-17
   ```

#### **Paso 6.(Opcional)** SQL Developer se iniciará automáticamente una vez que se proporcione la ubicación del JDK.

<br/><br/>


## Resultado esperado

La imagen representa el entorno inicial de Oracle SQL Developer (versión 22.2.1.234.1810) con una conexión disponible a la base de datos Oracle, lista para ejecutar consultas o administrar objetos de la base de datos.

![SQLDeveloper](../images/img23.jpg)
