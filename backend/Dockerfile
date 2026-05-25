FROM golang:1.17-alpine AS build

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /server ./cmd/api

FROM alpine:3.22 AS run

RUN addgroup -S app && adduser -S app -G app

COPY --from=build /server /server

USER app

EXPOSE 8081

CMD ["/server"]
