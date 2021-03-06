FROM node:12.20.1-alpine3.12 as builder

RUN apk update

# Need JDK for the openapi-generator that is part of the build process
RUN apk add --no-cache openjdk8=8.275.01-r0

# Need git because some of our npm depedencies might be coming
# straight from github instead of being an npm package on npmjs.com.
RUN apk add --no-cache git

# Some install scripts of the npm package fabric-network need python
RUN apk add --no-cache python3 py3-pip

RUN npm install modclean -g

WORKDIR /
RUN mkdir /app/
WORKDIR /app/
COPY ./ ./
RUN npm ci
RUN ./node_modules/.bin/lerna clean --yes
RUN ./node_modules/.bin/lerna bootstrap
RUN npm run build:dev:backend
RUN npm run webpack:dev:web
RUN npm run build:dev:frontend -- --scope='@hyperledger/cactus-example-supply-chain-frontend'
RUN ./node_modules/.bin/lerna clean --yes
RUN ./node_modules/.bin/lerna bootstrap -- --production --no-optional

# RUN modclean --run --patterns="default:*" --path ./examples/cactus-example-supply-chain-backend/
# RUN modclean --run --patterns="default:*" --path ./examples/cactus-example-supply-chain-business-logic-plugin/
# RUN modclean --run --patterns="default:*" --path ./examples/cactus-example-supply-chain-frontend/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-api-client/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-cmd-api-server/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-cockpit/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-common/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-core-api/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-core/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-plugin-consortium-manual/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-plugin-keychain-memory/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-plugin-keychain-vault/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-plugin-ledger-connector-besu/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-plugin-ledger-connector-fabric/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-plugin-ledger-connector-quorum/
# RUN modclean --run --patterns="default:*" --path ./packages/cactus-test-tooling/

RUN rm -rf ./packages/cactus-test-plugin*
RUN rm -rf ./packages/cactus-test-cmd*
RUN rm -rf ./packages/cactus-test-api*
RUN rm -rf ./node_modules/

FROM docker:20.10.2-dind-rootless as runner

USER root

RUN apk update

# Add bash for convenience, sh has less features
RUN apk add --no-cache bash=5.0.17-r0

# Need curl for healthchecks
RUN apk add --no-cache curl=7.69.1-r3

# The file binary is used to inspect exectubles when debugging container image issues
RUN apk add --no-cache file=5.38-r0

RUN apk add --no-cache nodejs=12.20.1-r0
RUN apk add --no-cache npm=12.20.1-r0

RUN apk add --no-cache ca-certificates=20191127-r4
RUN apk add --no-cache tzdata

# Install supervisord because we need to run the docker daemon and also the fabric network
# meaning that we have multiple processes to run.
RUN apk add --no-cache supervisor=4.2.0-r0

ARG APP=/usr/src/app/

ENV TZ=Etc/UTC
ENV APP_USER=appuser

RUN addgroup --system $APP_USER
RUN adduser --system $APP_USER -G $APP_USER
RUN addgroup $APP_USER rootless
RUN mkdir -p ${APP}

COPY --chown=$APP_USER:$APP_USER --from=builder /app/ ${APP}

RUN mkdir -p "${APP}/log/"
RUN chown -R $APP_USER:$APP_USER "${APP}/log/"

# TODO: Can we hack it together so that the whole thing works rootless?
# USER $APP_USER

WORKDIR ${APP}

COPY --chown=${APP_USER}:${APP_USER} ./examples/supply-chain-app/healthcheck.sh /

ENV CACTUS_NODE_ID=-
ENV CONSORTIUM_ID=-
ENV KEY_PAIR_PEM=-
ENV COCKPIT_WWW_ROOT=examples/cactus-example-supply-chain-frontend/www/
ENV COCKPIT_TLS_ENABLED=false
ENV COCKPIT_CORS_DOMAIN_CSV=\*
ENV COCKPIT_MTLS_ENABLED=false
ENV COCKPIT_TLS_CERT_PEM=-
ENV COCKPIT_TLS_KEY_PEM=-
ENV COCKPIT_TLS_CLIENT_CA_PEM=-
ENV COCKPIT_HOST=0.0.0.0
ENV COCKPIT_PORT=3000
ENV API_MTLS_ENABLED=false
ENV API_TLS_ENABLED=false
ENV API_CORS_DOMAIN_CSV=\*
ENV API_TLS_CERT_PEM=-
ENV API_TLS_CLIENT_CA_PEM=-
ENV API_TLS_KEY_PEM=-
ENV API_HOST=0.0.0.0
ENV API_PORT=4000
ENV LOG_LEVEL=TRACE

COPY examples/supply-chain-app/supervisord.conf /etc/supervisord.conf

# supervisord web ui/dashboard
EXPOSE 9001
# API #1
EXPOSE 4000
# API #2
EXPOSE 4100
# GUI #1
EXPOSE 3000
# GUI #2
EXPOSE 3100
# API #3
EXPOSE 4200
# GUI #3
EXPOSE 3200

# Extend the parent image's entrypoint
# https://superuser.com/questions/1459466/can-i-add-an-additional-docker-entrypoint-script
ENTRYPOINT ["/usr/bin/supervisord"]
CMD ["--configuration", "/etc/supervisord.conf", "--nodaemon"]
HEALTHCHECK --interval=1s --timeout=5s --start-period=20s --retries=250 \
    CMD /usr/src/app/examples/supply-chain-app/healthcheck.sh
