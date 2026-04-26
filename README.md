# gemini-captcha-solver

A Go library that solves image-based CAPTCHAs using the Google Gemini API. Supports API key pooling for high-throughput usage.

## Installation

```bash
go get github.com/KilimcininKorOglu/gemini-captcha-solver
```

Requires Go 1.22 or later.

## Quick Start

```go
package main

import (
	"fmt"
	"os"

	captcha "github.com/KilimcininKorOglu/gemini-captcha-solver"
)

func main() {
	solver := captcha.New(captcha.Config{
		APIKey: os.Getenv("GEMINI_API_KEY"),
	})

	imageData, _ := os.ReadFile("captcha.jpg")

	code, err := solver.Solve(imageData)
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
	fmt.Println(code)
}
```

## API Key Pool

Distribute requests across multiple API keys to avoid rate limits. Keys are rotated round-robin; on HTTP 429, the solver instantly switches to the next available key.

```go
solver := captcha.New(captcha.Config{
	APIKeys: []string{
		os.Getenv("GEMINI_KEY_1"),
		os.Getenv("GEMINI_KEY_2"),
		os.Getenv("GEMINI_KEY_3"),
	},
})
```

The solver is safe for concurrent use from multiple goroutines.

## Configuration

| Field      | Type          | Default                | Description                           |
|------------|---------------|------------------------|---------------------------------------|
| APIKey     | string        |                        | Single API key                        |
| APIKeys    | []string      |                        | Key pool (round-robin, takes priority over APIKey) |
| Model      | string        | `gemini-2.5-flash`     | Gemini model name                     |
| Prompt     | string        | Generic CAPTCHA prompt | Custom prompt sent to Gemini          |
| MaxRetries | int           | 5                      | Max retry attempts on rate limit      |
| Backoff    | time.Duration | 15s                    | Initial backoff duration              |

## Rate Limit Handling

1. Rotate to the next key in pool (instant, no delay)
2. If all keys are rate limited, respect `Retry-After` header from the API response
3. If no `Retry-After` header is present, apply exponential backoff (15s, 30s, 60s, 120s, 120s)

## How It Works

1. Base64-encodes the CAPTCHA image
2. Sends it to the Gemini API with a text prompt asking to read the characters
3. Parses the response, strips non-alphanumeric characters, validates length (4--8 chars)
4. Returns the cleaned CAPTCHA text

Temperature is set to 0 for deterministic output.

## Getting an API Key

Get a free Gemini API key at [Google AI Studio](https://aistudio.google.com/app/apikey).

## License

MIT
