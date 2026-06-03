# ─────────────────────────────────────────────
# STAGE 1: Build — compila el JAR con Maven
# ─────────────────────────────────────────────
FROM eclipse-temurin:17-jdk AS builder

WORKDIR /app

# Copiar primero pom.xml y wrapper para cachear dependencias
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN chmod +x mvnw && ./mvnw dependency:go-offline -q

# Copiar código fuente y compilar (sin tests)
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# ─────────────────────────────────────────────
# STAGE 2: Runtime — imagen mínima sin JDK
# ─────────────────────────────────────────────
FROM eclipse-temurin:17-jre AS runtime

# Crear usuario no-root para seguridad (mínimo privilegio)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copiar solo el JAR generado en el stage anterior
COPY --from=builder /app/target/*.jar app.jar

# Cambiar propietario del directorio al usuario no-root
RUN chown -R appuser:appgroup /app

# Ejecutar como usuario no-root
USER appuser

# Exponer el puerto del servicio (configurado en application.properties)
EXPOSE 8081

# Variables de entorno con valores por defecto (se sobreescriben en docker-compose o EC2)
ENV DB_ENDPOINT=localhost \
    DB_PORT=3306 \
    DB_NAME=despachos_db \
    DB_USERNAME=app_user \
    DB_PASSWORD=change_me

# Health check para Docker y EC2
HEALTHCHECK --interval=30s --timeout=10s --start-period=45s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8081/swagger-ui.html || exit 1

# Punto de entrada
ENTRYPOINT ["java", "-jar", "app.jar"]
