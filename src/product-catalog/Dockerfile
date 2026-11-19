FROM golang:1.22-alpine AS builder
WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 go build -o /bin/server .

FROM alpine
WORKDIR /app

COPY --from=builder /app/products ./products
COPY --from=builder /bin/server .

ENV PRODUCT_CATALOG_PORT=8088
CMD ["./server"]
