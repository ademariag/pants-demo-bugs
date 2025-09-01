ARG BASE_IMAGE=:base

FROM ${BASE_IMAGE}
RUN if command -v apt-get >/dev/null 2>&1; then apt-get update && apt-get install -y python3 python3-pip; elif command -v apk >/dev/null 2>&1; then apk add --no-cache python3 py3-pip; fi