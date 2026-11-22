# wraith-go

Idiomatic Go client for the [Wraith](https://github.com/Miracle656/wraith)
Soroban token-transfer indexer REST API.

The client is split into **one package per resource** — `transfers`,
`accounts`, `assets`, `nfts`, `webhooks`, `status` — all sharing a small
transport in the root `wraith` package. Amounts are returned as `string` to
preserve full i128 precision.

## Install

```bash
go get github.com/Miracle656/wraith/clients/go
```

## Usage

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/Miracle656/wraith/clients/go/wraith"
	"github.com/Miracle656/wraith/clients/go/transfers"
	"github.com/Miracle656/wraith/clients/go/assets"
)

func main() {
	c := wraith.New("https://wraith.example.com")
	ctx := context.Background()

	// Transfers received by an address.
	tc := transfers.New(c)
	page, err := tc.Incoming(ctx, "GABC...", &transfers.ListParams{
		Limit: wraith.IntPtr(100),
	})
	if err != nil {
		log.Fatal(err)
	}
	for _, t := range page.Transfers {
		fmt.Println(t.ContractID, t.Amount, t.EventType)
	}
	// Check whether another page exists before fetching more.
	if page.HasMore && page.NextCursor != nil {
		fmt.Println("next cursor:", *page.NextCursor)
	}

	// Most-active assets.
	ac := assets.New(c)
	pop, err := ac.PopularAssets(ctx, &assets.PopularParams{Window: "24h", By: "volume"})
	if err != nil {
		log.Fatal(err)
	}
	for _, a := range pop.Assets {
		fmt.Println(a.ContractID, a.TransferCount, a.Volume)
	}
}
```

A non-2xx response is returned as a `*wraith.APIError`:

```go
page, err := tc.Incoming(ctx, "GABC...", nil)
var apiErr *wraith.APIError
if errors.As(err, &apiErr) {
	fmt.Println(apiErr.StatusCode, apiErr.Message)
}
```

## Packages

| Package     | Constructor             | List methods                                  |
| ----------- | ----------------------- | --------------------------------------------- |
| `wraith`    | `wraith.New(baseURL)`   | shared transport, `APIError`, options         |
| `transfers` | `transfers.New(c)`      | `Incoming`, `Outgoing`, `ForAddress`, `ByTx`  |
| `accounts`  | `accounts.New(c)`       | `Summary`, `Transfers`                        |
| `assets`    | `assets.New(c)`         | `PopularAssets`                               |
| `nfts`      | `nfts.New(c)`           | `Transfers`                                   |
| `webhooks`  | `webhooks.New(c)`       | `List`, `Deliveries`                          |
| `status`    | `status.New(c)`         | `Get`                                         |

Options: `wraith.WithHTTPClient(*http.Client)` and `wraith.WithHeader(k, v)`.

## Breaking changes

### `transfers.Page.Total` is now `*int`

The API only returns `total` when the caller passes `includeTotal=true`. An
absent field is meaningfully different from zero — returning 0 previously
meant both "not requested" and "no results". Code that read `page.Total`
unconditionally should now check for nil:

```go
page, _ := tc.Incoming(ctx, addr, &transfers.ListParams{IncludeTotal: true})
if page.Total != nil {
    fmt.Println("total:", *page.Total)
}
```

### `transfers.Page.HasMore` added

The primary pagination signal. The API always returns `hasMore`; `Total` is
only present when explicitly requested and is slower to compute.

### `webhooks.Subscription.Filter` is now `*WebhookFilter`

Previously `*string`, which could not decode the actual filter object the API
returns. Any subscription with a filter set would fail to decode before this
fix. The new type is:

```go
type WebhookFilter struct {
    Contract  *string `json:"contract,omitempty"`
    From      *string `json:"from,omitempty"`
    To        *string `json:"to,omitempty"`
    MinAmount *string `json:"min_amount,omitempty"`
}
```

A `nil` `Filter` on a `Subscription` means no filter is configured (the API
returns `"filter": null`).

### `status.Status` now includes `Status`, `Network`, and `Networks`

The `/status` response has always included these fields; the struct was
missing them. `LastIndexedLedger`, `LatestLedger`, and `LagLedgers` are now
`*int64` (nullable) to match the API spec — the server can return `null` for
these when the indexer has not committed a ledger yet.

## Development

```bash
cd clients/go
go test ./... -cover
go vet ./...
gofmt -l .
```
