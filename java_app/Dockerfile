# Choisis une image JRE légère (alpine)
FROM eclipse-temurin:17-jre-alpine

# Copie le bytecode compilé (Main.class) dans le conteneur
COPY Main.class /app/Main.class
COPY --from=0 /usr/src/Main.class .

# Définit le répertoire de travail
WORKDIR /app

# Commande par défaut : exécute la classe Main
CMD ["java", "Main"]
