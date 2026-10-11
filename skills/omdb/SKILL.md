---
name: omdb
description: Use when the user wants to look up movie, film, or TV show information — search by title, query by IMDb ID, get ratings (IMDb, Rotten Tomatoes, Metacritic), plot, poster, or metadata via the OMDb API.
trigger: user wants to look up movie/TV show information
tags: [omdb, movies, films, tv, api, imdb, rotten-tomatoes, metacritic, metadata, ratings]
---

# OMDb Skill

Query the Open Movie Database (OMDb) API for movie, series, and episode metadata.

## Configuration

This skill reads configuration from environment variables.

### Environment variables

| Variable | Required | Secret | Purpose |
|----------|----------|--------|---------|
| `OMDB_API_KEY` | Yes | Yes | OMDb API key. **Never read, echo, print, or log.** Use `$OMDB_API_KEY` only. |

### Preflight check

```bash
if [ -z "$OMDB_API_KEY" ]; then echo "MISSING: OMDB_API_KEY"; else echo "OMDB_API_KEY set"; fi
```

If MISSING: **stop and ask the user to set it.** Do not proceed.

### The Secrets Law

For `OMDB_API_KEY`: **NEVER read, echo, print, log, or otherwise expose the value.** Use only the `$OMDB_API_KEY` placeholder.

- ✅ `curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix"`
- ❌ `echo "$OMDB_API_KEY"`
- ❌ `printenv OMDB_API_KEY`
- ❌ writing the value into a script, env file, or doc

## Common Operations

### By Title
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix"
```

### By IMDb ID
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&i=tt0133093"
```

### With Year
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix&y=1999"
```

### Full Plot
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix&plot=full"
```

### By Type (movie/series/episode)
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=Breaking+Bad&type=series"
```

### Search (paginated)
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&s=Matrix&page=1"
```

### JSONP (with callback)
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix&callback=myHandler"
```

### XML Response
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix&r=xml"
```

### Season Episodes
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=Breaking+Bad&Season=1"
```

### Single Episode
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=Breaking+Bad&Season=1&Episode=1"
```

## Parameters Reference

| Param | Required | Values | Default | Description |
|-------|----------|--------|---------|-------------|
| `i` | No* | IMDb ID | — | Lookup by ID (e.g. tt1285016) |
| `t` | No* | string | — | Lookup by title |
| `s` | Yes (search) | string | — | Search by title |
| `type` | No | movie, series, episode | — | Filter result type |
| `y` | No | year | — | Year of release |
| `plot` | No | short, full | short | Plot length |
| `r` | No | json, xml | json | Response format |
| `page` | No | 1-100 | 1 | Search page number |
| `Season` | No | number | — | Season number (returns all episodes) |
| `Episode` | No | number | — | Episode number (requires Season) |
| `callback` | No | string | — | JSONP callback name |

*At least one of `i` or `t` is required for single-title lookup.

## Response Structures

### Single title/movie lookup (`t` or `i`)
```json
{
  "Title": "The Matrix",
  "Year": "1999",
  "Rated": "R",
  "Released": "31 Mar 1999",
  "Runtime": "136 min",
  "Genre": "Action, Sci-Fi",
  "Director": "Lana Wachowski, Lilly Wachowski",
  "Writer": "Lilly Wachowski, Lana Wachowski",
  "Actors": "Keanu Reeves, Laurence Fishburne, Carrie-Anne Moss",
  "Plot": "When a beautiful stranger leads computer hacker Neo to a forbidding underworld...",
  "Language": "English",
  "Country": "United States, Australia",
  "Awards": "Won 4 Oscars. 42 wins & 52 nominations total",
  "Poster": "https://m.media-amazon.com/images/...jpg",
  "Ratings": [
    {"Source": "Internet Movie Database", "Value": "8.7/10"},
    {"Source": "Rotten Tomatoes", "Value": "83%"},
    {"Source": "Metacritic", "Value": "73/100"}
  ],
  "Metascore": "73",
  "imdbRating": "8.7",
  "imdbVotes": "2,243,093",
  "imdbID": "tt0133093",
  "Type": "movie",
  "DVD": "N/A",
  "BoxOffice": "$177,559,005",
  "Production": "N/A",
  "Website": "N/A",
  "Response": "True"
}
```

### Series lookup (`type=series`)
```json
{
  "Title": "Breaking Bad",
  "Year": "2008–2013",
  "Rated": "TV-MA",
  "Released": "20 Jan 2008",
  "Runtime": "49 min",
  "Genre": "Crime, Drama, Thriller",
  "Director": "N/A",
  "Writer": "Vince Gilligan",
  "Actors": "Bryan Cranston, Aaron Paul, Anna Gunn",
  "Plot": "A chemistry teacher diagnosed with inoperable lung cancer...",
  "Awards": "Won 16 Primetime Emmys. 172 wins & 269 nominations total",
  "Ratings": [
    {"Source": "Internet Movie Database", "Value": "9.5/10"},
    {"Source": "Rotten Tomatoes", "Value": "96%"}
  ],
  "imdbRating": "9.5",
  "imdbVotes": "2,602,845",
  "imdbID": "tt0903747",
  "Type": "series",
  "totalSeasons": "5",
  "Response": "True"
}
```

### Search results (`s`)
```json
{
  "Search": [
    {"Title": "The Matrix", "Year": "1999", "imdbID": "tt0133093", "Type": "movie", "Poster": "https://..."},
    {"Title": "The Matrix Reloaded", "Year": "2003", "imdbID": "tt0234215", "Type": "movie", "Poster": "https://..."}
  ],
  "totalResults": "156",
  "Response": "True"
}
```

### Season episodes (`Season=1`)
```json
{
  "Title": "Breaking Bad",
  "Season": "1",
  "totalSeasons": "5",
  "Episodes": [
    {"Title": "Pilot", "Released": "2008-01-20", "Episode": "1", "imdbRating": "9.0", "imdbID": "tt0959621"},
    {"Title": "Cat's in the Bag...", "Released": "2008-01-27", "Episode": "2", "imdbRating": "8.6", "imdbID": "tt1054724"}
  ],
  "Response": "True"
}
```

### Single episode (`Season=1&Episode=1`)
Same shape as movie lookup but with `Type: "episode"`, `seriesID`, and `Season`/`Episode` fields.

### Error response
```json
{
  "Response": "False",
  "Error": "Movie not found!"
}
```

## Parsing Results with jq

### Single title — extract key info
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix" | \
  jq '{title: .Title, year: .Year, rated: .Rated, runtime: .Runtime, genre: .Genre, imdb: .imdbRating, metascore: .Metascore}'
```

### Ratings breakdown
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix" | \
  jq '.Ratings[] | "\(.Source): \(.Value)"'
```

### Search results — titles and years
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&s=Matrix" | \
  jq '.Search[] | "\(.Title) (\(.Year)) — \(.imdbID)"'
```

### Search — total results count
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&s=Matrix" | \
  jq '{total: .totalResults, results: [.Search[] | {title: .Title, year: .Year, id: .imdbID}]}'
```

### Filter by type from search results
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&s=Star+Wars" | \
  jq '[.Search[] | select(.Type == "movie")]'
```

### Check if response succeeded
```bash
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=NonExistentMovie" | \
  jq '.Response'
# "False" — check .Error for the reason
```

## Quick Reference

```bash
# Lookup by title
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix"
# Lookup by IMDb ID
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&i=tt0133093"
# Search
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&s=Matrix"
# Season episodes
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=Breaking+Bad&Season=1"
# Single episode
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=Breaking+Bad&Season=1&Episode=1"
# Full plot
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=The+Matrix&plot=full"
# Filter by type
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&t=Breaking+Bad&type=series"
# Paginated search
curl -s "https://www.omdbapi.com/?apikey=$OMDB_API_KEY&s=Matrix&page=2"
```

## Poster API

Patron-only feature. Requires separate API key.

```bash
curl -s "https://img.omdbapi.com/?apikey=$OMDB_API_KEY&i=tt0133093"
```

## Typical Response Fields

| Field | Description |
|-------|-------------|
| `Title` | Movie/series title |
| `Year` | Release year |
| `Rated` | Rating (e.g. PG-13, R) |
| `Released` | Release date |
| `Runtime` | Duration |
| `Genre` | Comma-separated genres |
| `Director` / `Writer` / `Actors` | Credits |
| `Plot` | Synopsis |
| `Language` / `Country` | Origin info |
| `Awards` | Award summary |
| `Ratings` | Array of {Source, Value} |
| `Metascore` / `imdbRating` / `imdbVotes` | Scores |
| `imdbID` | IMDb identifier |
| `Type` | movie, series, or episode |
| `totalSeasons` | For series |
| `Response` | "True" or "False" |
| `Error` | Error message (if Response=False) |

## Troubleshooting

- **`{"Response":"False","Error":"Movie not found!"}`**: Title not in OMDb — try different wording, add year, or search with `s` instead of `t`
- **`{"Response":"False","Error":"Incorrect IMDb ID."}`**: Check the ID format (should be `tt` + 7-8 digits)
- **`{"Response":"False","Error":"API key invalid."}`**: Key is wrong or revoked — get a new one at https://www.omdbapi.com/apikey.aspx
- **Search returns no results**: Try fewer words, omit year, or use a different title variant
- **Free tier limits**: 1,000 requests/day per key
