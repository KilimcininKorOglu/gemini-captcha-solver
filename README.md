# gemini-captcha-solver

A Go library that solves image-based CAPTCHAs using the Google Gemini API. Supports API key pooling for high-throughput usage.

## Installation

```bash
go get github.com/KilimcininKorOglu/gemini-captcha-solver
```

Requires Go 1.26 or later.

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

Distribute requests across multiple API keys to avoid rate limits. Keys rotate round-robin. On HTTP 429, the rate-limited key enters a cooldown period while the remaining keys continue serving requests. If all keys are cooling down, the solver waits for the earliest one to become available.

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

| Field      | Type     | Default                 | Description                                                        |
|------------|----------|-------------------------|--------------------------------------------------------------------|
| APIKey     | string   |                         | Single API key                                                     |
| APIKeys    | []string |                         | Key pool (round-robin, takes priority over APIKey)                 |
| Model      | string   | `gemini-2.5-flash-lite` | Gemini model name                                                  |
| Prompt     | string   | Generic CAPTCHA prompt  | Custom prompt sent to Gemini                                       |
| MaxRetries | int      | 5                       | Max attempts for non-rate-limit errors (429 retries are unlimited) |

## Rate Limit Handling

1. On HTTP 429, the rate-limited key enters a per-key cooldown
2. Cooldown duration comes from the `Retry-After` response header; if absent, falls back to 60 seconds
3. Other keys in the pool remain available and are tried immediately
4. If all keys are cooling down, the solver sleeps until the earliest cooldown expires
5. A hard deadline of 5 minutes caps total `Solve` wall time regardless of retries

## How It Works

1. Base64-encodes the CAPTCHA image
2. Sends it to the Gemini API with a text prompt asking to read the characters
3. Parses the response, lowercases the text, strips non-alphanumeric characters, validates length (4-8 chars)
4. Returns the cleaned lowercase CAPTCHA text

Temperature is set to 0 for deterministic output.

## Getting an API Key

Get a free Gemini API key at [Google AI Studio](https://aistudio.google.com/app/apikey).

## License

MIT
