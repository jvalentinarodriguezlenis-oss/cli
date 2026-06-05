# GitHub CLI API Package Documentation

The `api` package provides utilities for making requests to the GitHub API with support for both GraphQL and REST endpoints. This document explains the core components and usage patterns.

## Overview

The GitHub CLI API package (`github.com/cli/cli/v2/api`) wraps the `go-gh` library and provides:

- **GraphQL support** - Query and mutation operations
- **REST API support** - GET, POST, PUT, DELETE operations with pagination
- **Error handling** - Custom error types with OAuth scope suggestions
- **HTTP client management** - Centralized HTTP transport handling
- **Authentication** - Automatic token and header management

## Core Components

### Client

The `Client` struct is the main entry point for making API calls.

```go
type Client struct {
	http *http.Client
}

func NewClientFromHTTP(httpClient *http.Client) *Client
```

**Methods:**
- `GraphQL(hostname, query, variables, data)` - Execute a GraphQL query
- `Query(hostname, name, query, variables)` - Named GraphQL query
- `Mutate(hostname, name, mutation, variables)` - GraphQL mutation
- `QueryWithContext(ctx, hostname, name, query, variables)` - Context-aware query
- `REST(hostname, method, path, body, data)` - REST API request
- `RESTWithNext(hostname, method, path, body, data)` - REST request with pagination
- `HTTP()` - Access underlying HTTP client

### Error Types

#### GraphQLError

Represents errors returned from GraphQL operations. Wraps `go-gh`'s `GraphQLError` type.

```go
type GraphQLError struct {
	*ghAPI.GraphQLError
}
```

#### HTTPError

Represents HTTP errors with OAuth scope information.

```go
type HTTPError struct {
	*ghAPI.HTTPError
	scopesSuggestion string
}

func (err HTTPError) ScopesSuggestion() string
```

Provides helpful messages when API calls fail due to missing OAuth scopes.

## Usage Examples

### GraphQL Query

```go
type QueryData struct {
	Repository struct {
		Name        string
		Description string
	} `graphql:"repository(owner:$owner, name:$name)"`
}

query := `
  query GetRepo($owner: String!, $name: String!) {
    repository(owner: $owner, name: $name) {
      name
      description
    }
  }
`

client := api.NewClientFromHTTP(httpClient)
data := &QueryData{}
err := client.GraphQL("github.com", query, 
	map[string]interface{}{
		"owner": "cli",
		"name": "cli",
	}, 
	data)
```

### GraphQL Mutation

```go
type MutationData struct {
	AddReaction struct {
		Subject struct {
			ReactionCount int
		}
	}
}

mutation := `
  mutation AddReaction($input: AddReactionInput!) {
    addReaction(input: $input) {
      subject {
        reactionCount
      }
    }
  }
`

data := &MutationData{}
err := client.Mutate("github.com", "AddReactionMutation",
	data,
	map[string]interface{}{
		"input": map[string]interface{}{
			"subjectId": "MDEyOklzc3VlMQ==",
			"content": "THUMBS_UP",
		},
	})
```

### REST API Request

```go
type Repository struct {
	Name        string `json:"name"`
	Description string `json:"description"`
	StarCount   int    `json:"stargazers_count"`
}

repo := &Repository{}
err := client.REST("github.com", "GET", 
	"repos/cli/cli", 
	nil, 
	repo)
```

### REST API with Pagination

```go
type Issue struct {
	Number int    `json:"number"`
	Title  string `json:"title"`
}

var issues []*Issue
nextURL := "repos/cli/cli/issues?per_page=30"

for nextURL != "" {
	var pageIssues []*Issue
	var err error
	nextURL, err = client.RESTWithNext("github.com", "GET", 
		nextURL, 
		nil, 
		&pageIssues)
	if err != nil {
		return err
	}
	issues = append(issues, pageIssues...)
}
```

## API Constants

```go
const (
	apiVersion      = "X-GitHub-Api-Version"
	apiVersionValue = "2022-11-28"
	authorization   = "Authorization"
	cacheTTL        = "X-GH-CACHE-TTL"
	graphqlFeatures = "GraphQL-Features"
	features        = "merge_queue"
	userAgent       = "User-Agent"
)
```

## Error Handling

### Handling GraphQL Errors

```go
err := client.GraphQL(hostname, query, variables, &data)
if err != nil {
	var gqlErr api.GraphQLError
	if errors.As(err, &gqlErr) {
		// Handle GraphQL-specific error
		fmt.Printf("GraphQL Error: %v\n", gqlErr)
	}
}
```

### Handling HTTP Errors

```go
err := client.REST(hostname, "GET", path, nil, &data)
if err != nil {
	var httpErr api.HTTPError
	if errors.As(err, &httpErr) {
		fmt.Printf("HTTP Status: %d\n", httpErr.StatusCode)
		
		// Check if additional scopes are needed
		suggestion := httpErr.ScopesSuggestion()
		if suggestion != "" {
			fmt.Println(suggestion)
		}
	}
}
```

## OAuth Scope Management

### Scope Suggestions

The API automatically generates OAuth scope suggestions when requests fail due to insufficient permissions.

```go
// Helper function to get scopes needed for an endpoint
func ScopesSuggestion(resp *http.Response) string
```

### Adding Endpoint-Specific Scopes

Some endpoints may not explicitly declare their required scopes. Use `EndpointNeedsScopes` to add them:

```go
resp, _ := http.Get(url)
api.EndpointNeedsScopes(resp, "repo:status")
```

## Scope Hierarchy

The API understands scope hierarchies:
- `repo` implies: `repo:status`, `repo_deployment`, `public_repo`, `repo:invite`, `security_events`
- `user` implies: `read:user`, `user:email`, `user:follow`
- `codespace` implies: `codespace:secrets`
- `admin:*` implies: `read:*` and `write:*`
- `write:*` implies: `read:*`

## Query Patterns

### Named Queries

The package supports named GraphQL queries for better error reporting:

```go
type PullRequestData struct {
	Repository struct {
		PullRequests struct {
			Nodes []struct {
				Number int
				Title  string
			}
		} `graphql:"pullRequests(first:10, states:OPEN)"`
	} `graphql:"repository(owner:$owner, name:$name)"`
}

data := &PullRequestData{}
err := client.Query("github.com", "GetOpenPullRequests", data, 
	map[string]interface{}{
		"owner": "cli",
		"name": "cli",
	})
```

## Advanced HTTP Configuration

### Custom HTTP Client

```go
httpClient := &http.Client{
	Timeout: 30 * time.Second,
	Transport: &http.Transport{
		MaxIdleConns: 10,
	},
}

client := api.NewClientFromHTTP(httpClient)
```

### Client Options

The package uses internal `clientOptions` to configure:
- Authorization headers (handled by transport)
- API version headers
- Custom headers
- Host configuration
- Transport customization

## Pagination

### Link Header Pagination

The `RESTWithNext` method handles RFC 5988 Link header pagination automatically:

```go
// Link header format: <url>; rel="next"
nextURL, err := client.RESTWithNext(hostname, method, path, body, &data)
```

The package parses Link headers and extracts the "next" URL for fetching subsequent pages.

## Testing

### Mocking the Client

For unit tests, you can mock the HTTP client:

```go
import "github.com/cli/cli/v2/pkg/httpmock"

reg := &httpmock.Registry{}
defer reg.Verify(t)

reg.Register(
	httpmock.GraphQL(`query GetRepo\b`),
	httpmock.JSONResponse(expectedData),
)

client := api.NewClientFromHTTP(&http.Client{Transport: reg})
```

## Related Packages

- `api/export_pr.go` - PR-specific export functionality
- `api/export_repo.go` - Repository export helpers
- `api/http_client.go` - HTTP client initialization
- `api/queries_*.go` - GraphQL query definitions
- `api/query_builder.go` - GraphQL query building utilities

## Best Practices

1. **Reuse clients** - Create one client per session, not per request
2. **Handle errors properly** - Always check for both GraphQL and HTTP errors
3. **Use context** - Prefer `QueryWithContext` for cancellable operations
4. **Pagination** - Use `RESTWithNext` for endpoints that return multiple pages
5. **Timeouts** - Configure appropriate timeouts on your HTTP client
6. **Scopes** - Request minimum necessary OAuth scopes; listen to scope suggestions
7. **Variables** - Use GraphQL variables instead of string interpolation to prevent injection attacks

## See Also

- [GitHub GraphQL API Documentation](https://docs.github.com/en/graphql)
- [GitHub REST API Documentation](https://docs.github.com/en/rest)
- [go-gh Package](https://github.com/cli/go-gh)
- [AGENTS.md](./AGENTS.md) - Architecture overview
