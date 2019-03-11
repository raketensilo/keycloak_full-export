FROM jboss/keycloak:4.3.0.Final

#ENV KEYCLOAK_USER=admin
#ENV KEYCLOAK_PASSWORD=admin

COPY keycloak_full-export.conf /opt/jboss/keycloak_full-export.conf
RUN /opt/jboss/keycloak/bin/jboss-cli.sh --file=keycloak_full-export.conf

EXPOSE 8080

ENTRYPOINT [ "/opt/jboss/docker-entrypoint.sh" ]

CMD ["-b", "0.0.0.0"]
