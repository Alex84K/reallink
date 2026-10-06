# Вывод намеренно короткий: при успехе — одна строка, при падении — только ошибки.
# Инструменты (oapi-codegen, goose) подключены через `go get -tool` и вызываются `go tool`.
GO      ?= go
S       ?=
N       ?=
DB_PATH ?= ./data/reallink.db
MIGRATIONS := internal/platform/db/migrations
FILTER  := grep -Ev '^(ok|\?) |no test files|^PASS$$'

.PHONY: gen run test test-slice lint migrate migrate-new build

gen:
	$(GO) tool oapi-codegen -config oapi-codegen.yaml ../api/openapi.yaml

run:
	$(GO) run ./cmd/server

test:
	@out=$$($(GO) test -race -count=1 ./... 2>&1); st=$$?; \
	if [ $$st -eq 0 ]; then echo "OK: all"; else echo "$$out" | $(FILTER); fi; exit $$st

test-slice:
	@test -n "$(S)" || { echo "usage: make test-slice S=<slice>"; exit 2; }
	@out=$$($(GO) test -count=1 ./internal/$(S)/... 2>&1); st=$$?; \
	if [ $$st -eq 0 ]; then echo "OK: $(S)"; else echo "$$out" | $(FILTER); fi; exit $$st

lint:
	golangci-lint run ./...

migrate:
	$(GO) tool goose -dir $(MIGRATIONS) sqlite3 $(DB_PATH) up

migrate-new:
	@test -n "$(N)" || { echo "usage: make migrate-new N=<name>"; exit 2; }
	$(GO) tool goose -dir $(MIGRATIONS) -s create $(N) sql

build:
	CGO_ENABLED=0 $(GO) build -o bin/server ./cmd/server
