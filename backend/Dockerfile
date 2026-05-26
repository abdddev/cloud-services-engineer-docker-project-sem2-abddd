FROM golang:1.25-alpine AS build

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /server ./cmd/api

FROM alpine:3.22 AS run

LABEL org.opencontainers.image.title="momo-store-backend" \
      org.opencontainers.image.description="Go API backend for momo-store"

RUN addgroup -S app \
    && adduser -S -G app -H -h /nonexistent app \
    && apk add --no-cache ca-certificates

COPY --from=build --chown=app:app /server /server

USER app

EXPOSE 8081

CMD ["/server"]
